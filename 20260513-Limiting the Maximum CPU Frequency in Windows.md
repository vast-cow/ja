---
pubDatetime: 2026-05-13T15:53:13+09:00
title: "Windows で CPU 周波数の最大値を制限する方法"
description: "背景、最初にハマった点、最終的に効いた設定を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Panasonic Let’s note CF-FV5USVCP の Windows 環境で、CPU の最大周波数を制限する方法を調べた。結論としては、通常の電源プランだけでなく、Windows の **オーバーレイ電源スキーム** と、Core Ultra 世代で出てくる **複数の CPU 効率クラス向け設定** までまとめて設定する必要があった。

今回、最終的に効いた設定は以下。

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

この例では、AC接続時もバッテリー駆動時も CPU 最大周波数を **1500 MHz** に制限している。

---

## 背景

CF-FV5USVCP は Intel Core Ultra 世代の CPU を搭載しており、P-core / E-core / Low Power E-core のように、複数種類のコアを持つ。従来のように「最大プロセッサの状態」を何％にするだけでは、期待通りに周波数が下がらないことがある。

Windows の `powercfg` には、CPU 周波数の最大値を MHz 単位で指定できる設定がある。

代表的なのは以下。

```text
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
```

今回の環境では、次のように確認できた。

```powershell
powercfg /qh SCHEME_CURRENT SUB_PROCESSOR | findstr /i "PROCFREQMAX PROCTHROTTLEMAX PERFEPP PERFBOOSTMODE"
```

出力には以下が含まれていた。

```text
PERFEPP
PERFEPP1
PERFEPP2
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
PROCTHROTTLEMAX
PROCTHROTTLEMAX1
PROCTHROTTLEMAX2
PERFBOOSTMODE
```

つまり、この環境では `PROCFREQMAX` だけでなく、`PROCFREQMAX1` と `PROCFREQMAX2` も設定対象に含めるべき、ということになる。

---

## 最初にハマった点

最初は、現在の電源プランに対して以下のように設定していた。

```powershell
powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX 3000
powercfg /setdcvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX 2200
powercfg /setactive SCHEME_CURRENT
```

しかし、実際には思ったように反映されなかった。

調べてみると、現在の電源プランは Panasonic 独自のものだった。

```powershell
powercfg /getactivescheme
```

出力：

```text
電源設定の GUID: a83ffe77-647e-45df-899e-cd3f4e2835a1  (パナソニックの電源管理)
```

さらに、Windows には通常の電源プランとは別に、電源モード用のオーバーレイがある。

確認したところ、`OVERLAY_SCHEME_MIN` 側には設定が入っていたが、`OVERLAY_SCHEME_MAX` 側は未設定だった。

```powershell
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX
```

出力：

```text
現在の AC 電源設定のインデックス: 0x00000000
現在の DC 電源設定のインデックス: 0x00000000
```

`0x00000000` は実質的に「制限なし」と見なせる。

一方、`OVERLAY_SCHEME_MIN` には値が入っていた。

```powershell
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX
```

出力：

```text
現在の AC 電源設定のインデックス: 0x00001194
現在の DC 電源設定のインデックス: 0x00000bb8
```

16進数を10進数に直すと、以下の意味になる。

```text
0x00001194 = 4500 MHz
0x00000bb8 = 3000 MHz
```

つまり、「より良いバッテリ寿命オーバーレイ」には制限が入っていたが、「最大パフォーマンス オーバーレイ」は未制限のままだった。

Windows の電源モードがパフォーマンス寄りになっていると、`OVERLAY_SCHEME_MAX` 側が使われる可能性がある。そのため、`SCHEME_CURRENT` だけ、あるいは `OVERLAY_SCHEME_MIN` だけを変更しても効かないことがある。

---

## 最終的に効いた設定

最終的には、以下を全部まとめて設定した。

対象スキーム：

```text
SCHEME_CURRENT
OVERLAY_SCHEME_MIN
OVERLAY_SCHEME_MAX
OVERLAY_SCHEME_HIGH
```

対象設定：

```text
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
```

実行したスクリプト：

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

これで、AC接続時・バッテリー駆動時ともに 1500 MHz 上限が効いた様子だった。

---

## 各コマンドの意味

### `$ACMHz` / `$DCMHz`

```powershell
$ACMHz = 1500
$DCMHz = 1500
```

AC電源接続時と、バッテリー駆動時の最大周波数を MHz 単位で指定する。

たとえば以下のように変えられる。

```powershell
$ACMHz = 3000
$DCMHz = 2200
```

発熱・ファン音を強く抑えたいなら 1500〜2200 MHz、ある程度性能を残したいなら 2500〜3500 MHz あたりが候補になる。

### `$Schemes`

```powershell
$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)
```

設定対象の電源スキームをまとめている。

| スキーム                  | 意味                 |
| --------------------- | ------------------ |
| `SCHEME_CURRENT`      | 現在有効な電源プラン         |
| `OVERLAY_SCHEME_MIN`  | 省電力寄りのオーバーレイ       |
| `OVERLAY_SCHEME_MAX`  | 最大パフォーマンス寄りのオーバーレイ |
| `OVERLAY_SCHEME_HIGH` | 高パフォーマンス系のオーバーレイ   |

今回の問題では、`SCHEME_CURRENT` だけでは不十分だった。`OVERLAY_SCHEME_MAX` が未制限のままだったため、パフォーマンス寄りの電源モードでは制限が効かなかった可能性が高い。

### `$FreqSettings`

```powershell
$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)
```

CPU 周波数上限の設定項目をまとめている。

Core Ultra 世代では、効率クラスごとに複数の設定が出ることがある。今回の環境では `PROCFREQMAX2` まで存在していたため、3つまとめて設定した。

### `setacvalueindex` / `setdcvalueindex`

```powershell
powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz
powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz
```

それぞれの意味は以下。

| コマンド                | 意味                    |
| ------------------- | --------------------- |
| `/setacvalueindex`  | AC電源接続時の値を設定          |
| `/setdcvalueindex`  | バッテリー駆動時の値を設定         |
| `SUB_PROCESSOR`     | プロセッサの電源管理            |
| `$setting`          | `PROCFREQMAX` などの設定項目 |
| `$ACMHz` / `$DCMHz` | 最大周波数 MHz             |

### `2>$null`

```powershell
2>$null
```

エラー出力を捨てる指定。

`PROCFREQMAX2` や `OVERLAY_SCHEME_HIGH` などは、環境によって存在しなかったり、参照できなかったりする。その場合にエラーが出るため、それを無視している。

ただし、最初の検証段階では `2>$null` を付けない方がよい。エラーが見えなくなるため、設定に失敗していることに気づきにくい。

検証時はこうする。

```powershell
powercfg /setacvalueindex OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX 1500
```

うまくいくことを確認してから、まとめて実行する段階で `2>$null` を付けるのがよい。

### `powercfg /setactive`

```powershell
powercfg /setactive SCHEME_CURRENT
```

設定後に現在の電源プランを再度有効化する。

設定変更を反映させる目的で最後に実行している。

---

## 設定確認方法

設定後、以下で確認する。

```powershell
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX1
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX2

powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX1
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX2

powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX1
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX2
```

1500 MHz に設定した場合、値は以下のようになる。

```text
0x000005dc
```

16進数 `0x000005dc` は 10進数で 1500。

2200 MHz なら以下。

```text
0x00000898
```

3000 MHz なら以下。

```text
0x00000bb8
```

4500 MHz なら以下。

```text
0x00001194
```

---

## 元に戻す方法

制限を解除したい場合は、`0` を設定する。

```powershell
$ACMHz = 0
$DCMHz = 0

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

`PROCFREQMAX` 系では、`0` は実質的に「制限なし」として扱われる。

---


## スリープ復帰後に自動で再適用する Python スクリプト

上記の `powercfg` 設定は基本的には保持されるが、環境によってはスリープや休止状態からの復帰後、Windows 側の電源モードやメーカー独自の電源管理によって挙動が戻ったように見えることがある。

そのため、復帰イベントを検知して CPU 周波数上限を自動で再適用する Python スクリプトも用意した。

このスクリプトは、起動時に一度 `powercfg` の設定を適用し、その後は Windows のサスペンド／レジューム通知を待ち受ける。スリープまたは休止状態から復帰したタイミングで、同じ `PROCFREQMAX` 系の設定を再度適用する。

```python
import ctypes
import ctypes.wintypes
import subprocess
import threading
import time
import atexit


if ctypes.sizeof(ctypes.c_void_p) == 8:
    ctypes.wintypes.LRESULT = ctypes.c_longlong
else:
    ctypes.wintypes.LRESULT = ctypes.c_long


# =========================
# Settings
# =========================

ACMHz = 1500
DCMHz = 1500

SCHEMES = [
    "SCHEME_CURRENT",
    "OVERLAY_SCHEME_MIN",
    "OVERLAY_SCHEME_MAX",
    "OVERLAY_SCHEME_HIGH",
]

FREQ_SETTINGS = [
    "PROCFREQMAX",
    "PROCFREQMAX1",
    "PROCFREQMAX2",
]


# =========================
# Windows constants
# =========================

CREATE_NO_WINDOW = 0x08000000

DEVICE_NOTIFY_CALLBACK = 0x00000002

PBT_APMSUSPEND = 0x0004
PBT_APMRESUMESUSPEND = 0x0007
PBT_APMRESUMEAUTOMATIC = 0x0012

WM_QUIT = 0x0012

CTRL_C_EVENT = 0
CTRL_BREAK_EVENT = 1
CTRL_CLOSE_EVENT = 2
CTRL_LOGOFF_EVENT = 5
CTRL_SHUTDOWN_EVENT = 6


# =========================
# ctypes definitions
# =========================

DEVICE_NOTIFY_CALLBACK_ROUTINE = ctypes.WINFUNCTYPE(
    ctypes.wintypes.ULONG,      # return
    ctypes.c_void_p,            # Context
    ctypes.wintypes.ULONG,      # Type
    ctypes.c_void_p,            # Setting
)


class DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS(ctypes.Structure):
    _fields_ = [
        ("Callback", DEVICE_NOTIFY_CALLBACK_ROUTINE),
        ("Context", ctypes.c_void_p),
    ]


class MSG(ctypes.Structure):
    _fields_ = [
        ("hwnd", ctypes.wintypes.HWND),
        ("message", ctypes.wintypes.UINT),
        ("wParam", ctypes.wintypes.WPARAM),
        ("lParam", ctypes.wintypes.LPARAM),
        ("time", ctypes.wintypes.DWORD),
        ("pt", ctypes.wintypes.POINT),
    ]


powrprof = ctypes.WinDLL("powrprof", use_last_error=True)
user32 = ctypes.WinDLL("user32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)


powrprof.PowerRegisterSuspendResumeNotification.restype = ctypes.wintypes.DWORD
powrprof.PowerRegisterSuspendResumeNotification.argtypes = [
    ctypes.wintypes.DWORD,
    ctypes.POINTER(DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS),
    ctypes.POINTER(ctypes.wintypes.HANDLE),
]

powrprof.PowerUnregisterSuspendResumeNotification.restype = ctypes.wintypes.DWORD
powrprof.PowerUnregisterSuspendResumeNotification.argtypes = [
    ctypes.wintypes.HANDLE,
]

user32.GetMessageW.restype = ctypes.wintypes.BOOL
user32.GetMessageW.argtypes = [
    ctypes.POINTER(MSG),
    ctypes.wintypes.HWND,
    ctypes.wintypes.UINT,
    ctypes.wintypes.UINT,
]

user32.TranslateMessage.restype = ctypes.wintypes.BOOL
user32.TranslateMessage.argtypes = [
    ctypes.POINTER(MSG),
]

user32.DispatchMessageW.restype = ctypes.wintypes.LRESULT
user32.DispatchMessageW.argtypes = [
    ctypes.POINTER(MSG),
]

user32.PostThreadMessageW.restype = ctypes.wintypes.BOOL
user32.PostThreadMessageW.argtypes = [
    ctypes.wintypes.DWORD,
    ctypes.wintypes.UINT,
    ctypes.wintypes.WPARAM,
    ctypes.wintypes.LPARAM,
]

kernel32.GetCurrentThreadId.restype = ctypes.wintypes.DWORD
kernel32.GetCurrentThreadId.argtypes = []

PHANDLER_ROUTINE = ctypes.WINFUNCTYPE(
    ctypes.wintypes.BOOL,
    ctypes.wintypes.DWORD,
)

kernel32.SetConsoleCtrlHandler.restype = ctypes.wintypes.BOOL
kernel32.SetConsoleCtrlHandler.argtypes = [
    PHANDLER_ROUTINE,
    ctypes.wintypes.BOOL,
]


# =========================
# global state
# =========================

_main_thread_id = ctypes.wintypes.DWORD(0)
_exiting = threading.Event()


# =========================
# powercfg execution
# =========================

def run_powercfg(args: list[str]) -> None:
    subprocess.run(
        ["powercfg", *args],
        stdin=subprocess.DEVNULL,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        creationflags=CREATE_NO_WINDOW,
        check=False,
    )


def apply_cpu_freq_limit() -> None:
    for scheme in SCHEMES:
        for setting in FREQ_SETTINGS:
            run_powercfg([
                "/setacvalueindex",
                scheme,
                "SUB_PROCESSOR",
                setting,
                str(ACMHz),
            ])

            run_powercfg([
                "/setdcvalueindex",
                scheme,
                "SUB_PROCESSOR",
                setting,
                str(DCMHz),
            ])

    run_powercfg([
        "/setactive",
        "SCHEME_CURRENT",
    ])


# =========================
# resume handling
# =========================

_apply_lock = threading.Lock()
_last_apply_time = 0.0


def apply_cpu_freq_limit_debounced() -> None:
    """
    PBT_APMRESUMEAUTOMATIC and PBT_APMRESUMESUSPEND
    may arrive consecutively, so suppress duplicate
    executions within a short interval.
    """
    global _last_apply_time

    if _exiting.is_set():
        return

    with _apply_lock:
        now = time.monotonic()

        if now - _last_apply_time < 5.0:
            return

        _last_apply_time = now
        apply_cpu_freq_limit()


def apply_async() -> None:
    if _exiting.is_set():
        return

    thread = threading.Thread(
        target=apply_cpu_freq_limit_debounced,
        daemon=True,
    )
    thread.start()


def power_notification_callback(context, event_type, setting) -> int:
    if event_type == PBT_APMRESUMESUSPEND:
        # Resume from sleep/hibernate initiated by user action
        apply_async()
        return 0

    if event_type == PBT_APMRESUMEAUTOMATIC:
        # Automatic resume.
        # Some environments only emit this event after hibernation.
        apply_async()
        return 0

    if event_type == PBT_APMSUSPEND:
        # Just before entering sleep/hibernate.
        # No action needed here.
        return 0

    return 0


# Keep a global reference so the callback is not garbage collected
_power_callback_ref = DEVICE_NOTIFY_CALLBACK_ROUTINE(power_notification_callback)
_notify_handle = ctypes.wintypes.HANDLE()


def register_power_notification() -> None:
    params = DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS()
    params.Callback = _power_callback_ref
    params.Context = None

    result = powrprof.PowerRegisterSuspendResumeNotification(
        DEVICE_NOTIFY_CALLBACK,
        ctypes.byref(params),
        ctypes.byref(_notify_handle),
    )

    if result != 0:
        raise ctypes.WinError(result)


def unregister_power_notification() -> None:
    if _notify_handle:
        powrprof.PowerUnregisterSuspendResumeNotification(_notify_handle)


# =========================
# Ctrl+C handling
# =========================

def request_exit() -> None:
    _exiting.set()

    if _main_thread_id.value:
        user32.PostThreadMessageW(
            _main_thread_id.value,
            WM_QUIT,
            0,
            0,
        )


def console_ctrl_handler(ctrl_type: int) -> bool:
    if ctrl_type in (
        CTRL_C_EVENT,
        CTRL_BREAK_EVENT,
        CTRL_CLOSE_EVENT,
        CTRL_LOGOFF_EVENT,
        CTRL_SHUTDOWN_EVENT,
    ):
        request_exit()
        return True

    return False


# Keep a global reference so the handler is not garbage collected
_console_ctrl_handler_ref = PHANDLER_ROUTINE(console_ctrl_handler)


def register_console_ctrl_handler() -> None:
    ok = kernel32.SetConsoleCtrlHandler(
        _console_ctrl_handler_ref,
        True,
    )

    if not ok:
        raise ctypes.WinError(ctypes.get_last_error())


# =========================
# message loop
# =========================

def message_loop() -> None:
    msg = MSG()

    while not _exiting.is_set():
        ret = user32.GetMessageW(ctypes.byref(msg), None, 0, 0)

        if ret == 0:
            break

        if ret == -1:
            raise ctypes.WinError(ctypes.get_last_error())

        user32.TranslateMessage(ctypes.byref(msg))
        user32.DispatchMessageW(ctypes.byref(msg))


def main() -> None:
    global _main_thread_id

    _main_thread_id = ctypes.wintypes.DWORD(kernel32.GetCurrentThreadId())

    atexit.register(unregister_power_notification)

    register_console_ctrl_handler()

    # Apply once at startup
    apply_cpu_freq_limit_debounced()

    register_power_notification()

    try:
        message_loop()
    except KeyboardInterrupt:
        request_exit()
    finally:
        unregister_power_notification()


if __name__ == "__main__":
    main()
```

### このスクリプトでやっていること

主な処理は以下。

```text
起動時に CPU 周波数上限を一度適用する
Windows のサスペンド／レジューム通知を登録する
スリープ／休止状態から復帰したら powercfg 設定を再適用する
Ctrl+C やシャットダウン時には通知登録を解除して終了する
```

復帰時に見ているイベントは以下。

| イベント | 意味 |
| --- | --- |
| `PBT_APMRESUMESUSPEND` | ユーザー操作によるスリープ／休止状態からの復帰 |
| `PBT_APMRESUMEAUTOMATIC` | 自動復帰。一部環境では休止状態からの復帰時にこちらだけ来ることがある |
| `PBT_APMSUSPEND` | スリープ／休止状態に入る直前 |

`PBT_APMRESUMESUSPEND` と `PBT_APMRESUMEAUTOMATIC` は連続して発生することがあるため、5秒以内の重複実行は抑制している。

### 実行方法

たとえば `apply_cpu_freq_limit_on_resume.py` という名前で保存し、管理者権限の PowerShell から実行する。

```powershell
python .\apply_cpu_freq_limit_on_resume.py
```

`powercfg` の設定変更には管理者権限が必要なので、通常の PowerShell ではなく、管理者として開いた PowerShell から起動する。

### スタートアップに登録する場合

常駐させたい場合は、タスク スケジューラに登録して、ログオン時に管理者権限で起動するようにしておく。

設定例は以下。

```text
トリガー: ログオン時
操作: python.exe apply_cpu_freq_limit_on_resume.py
権限: 最上位の特権で実行する
```

Python のパスが通っていない場合は、`python.exe` のフルパスを指定する。

例：

```text
C:\Users\<ユーザー名>\AppData\Local\Programs\Python\Python312\python.exe
```

スクリプトの引数には、保存した `.py` ファイルのフルパスを指定する。

```text
C:\path\to\apply_cpu_freq_limit_on_resume.py
```

### 値を変更したい場合

周波数上限を変えたい場合は、スクリプト冒頭の以下を変更する。

```python
ACMHz = 1500
DCMHz = 1500
```

たとえば、AC接続時は 3000 MHz、バッテリー駆動時は 2200 MHz にしたい場合はこうする。

```python
ACMHz = 3000
DCMHz = 2200
```

解除したい場合は、PowerShell 版と同じく `0` を指定する。

```python
ACMHz = 0
DCMHz = 0
```

### 注意点

このスクリプトは Windows 専用。`ctypes` で `powrprof.dll` や `user32.dll` を直接呼び出しているため、macOS や Linux では動かない。

また、スクリプトを起動している間だけ復帰イベントを待ち受ける。継続的に使うなら、タスク スケジューラでログオン時に自動起動するのが現実的。

## 追加でブーストも止めたい場合

周波数上限だけでなく、ターボブースト的な挙動も抑えたい場合は `PERFBOOSTMODE` を無効化する。

```powershell
$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

foreach ($scheme in $Schemes) {
  powercfg /setacvalueindex $scheme SUB_PROCESSOR PERFBOOSTMODE 0 2>$null
  powercfg /setdcvalueindex $scheme SUB_PROCESSOR PERFBOOSTMODE 0 2>$null
}

powercfg /setactive SCHEME_CURRENT
```

`PERFBOOSTMODE 0` はブースト無効を意味する。

ただし、今回の環境では `PROCFREQMAX` 系の設定だけでも効果が出た。

---

## 注意点

### 1. 管理者 PowerShell で実行する

`powercfg` の電源設定変更は、管理者権限の PowerShell で実行する。

### 2. 変数を定義し忘れない

今回一度ハマったのがこれ。

```powershell
$ACMHz
$DCMHz
```

を定義しないままスクリプトを実行すると、`powercfg` に値が渡らない。

さらに `2>$null` を付けていると、失敗してもエラーが見えない。

検証中は、まず以下のように明示的に値を入れて試すのがよい。

```powershell
powercfg /setacvalueindex OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX 1500
```

### 3. `PROCFREQMAX2` は環境によって出方が違う

今回の環境では `powercfg /qh` に `PROCFREQMAX2` が表示された。

ただし、個別に `powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX2` を実行しても、詳細が出ないケースがあった。

そのため、スクリプトでは `PROCFREQMAX2` も対象に入れつつ、エラーは `2>$null` で無視する形にした。

### 4. オーバーレイは非推奨警告が出る

実行時に以下の警告が出ることがある。

```text
警告: オーバーレイ スキームは非推奨となったため、今後はサポートされない可能性があります
```

今回の環境では、警告は出るが設定自体は有効だった。

将来の Windows バージョンで同じ方法が使えなくなる可能性はある。

---

## まとめ

CF-FV5USVCP のような Core Ultra 世代の Windows PC で CPU 最大周波数を制限する場合、単に `SCHEME_CURRENT` の `PROCFREQMAX` だけを設定しても効かないことがある。

今回効いたポイントは以下。

```text
SCHEME_CURRENT だけでなく overlay scheme も設定する
PROCFREQMAX だけでなく PROCFREQMAX1 / PROCFREQMAX2 も設定する
AC/DC の両方を設定する
最後に powercfg /setactive SCHEME_CURRENT を実行する
```

最終的に効いたスクリプトはこれ。

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

CF-FV5USVCP で発熱やファン音を抑えたい場合、まずは 1500〜2200 MHz 程度に制限して様子を見るのがよい。性能をもう少し残したい場合は、2500〜3000 MHz あたりに上げて調整する。
