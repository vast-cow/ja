---
pubDatetime: 2026-06-15T12:42:40+09:00
title: "プロセスごとのCPU・メモリ・I/O使用量の積算を確認するツール"
description: "目的、基本的な使い方、表示される主な項目を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 概要

Linux 上で動作するシンプルなプロセス監視ツールです。

通常の `top` は瞬時値を確認できますが、本ツールは CPU時間や読み書き の積算を表示できます。

## 目的

このツールの主な目的は、プロセスごとの負荷を簡単に確認することです。

CPU やメモリだけでなく、読み込み量、書き込み量、読み書き回数なども一覧で見られるため、負荷の原因を調べるときに役立ちます。

たとえば、次のような場面で使えます。

* CPU を多く使っているプロセスを確認したいとき
* メモリを多く使っているプロセスを確認したいとき
* ディスクやファイルへの読み書きが多いプロセスを探したいとき
* 一定時間のあいだに増えた I/O 量を確認したいとき

## 基本的な使い方

このツールは、Python スクリプトとして実行します。

```bash
python3 mytop.py
```

実行すると、端末上にプロセス一覧が表示されます。表示内容は一定間隔で更新され、各プロセスの CPU、メモリ、I/O 情報を確認できます。

## 表示される主な項目

画面には、プロセスごとの情報が列として表示されます。

主な項目は次のとおりです。

### PID

プロセス ID です。

### USER

そのプロセスを実行しているユーザーです。

### %CPU

直近の更新間隔で、そのプロセスが使用した CPU の割合です。

### %MEM

システム全体のメモリに対して、そのプロセスが使用しているメモリの割合です。

### DTIME+

このツールを起動してから、そのプロセスが使用した CPU 時間です。

### TIME+

プロセスがこれまでに使用した合計 CPU 時間です。

### RBYTES / WBYTES / RWBYTES

プロセスの読み込みバイト数、書き込みバイト数、読み書き合計バイト数です。

### RCHAR / WCHAR / RWCHAR

プロセスが読み書きした文字数のカウンターです。

### SYSCR / SYSCW / SYSCRW

読み込み・書き込みのシステムコール回数と、その合計です。

### COMMAND

実行されているコマンド名またはコマンドラインです。

## 並び替えの操作

実行中にキーを押すことで、表示の並び順を変更できます。

代表的なキーは次のとおりです。

| キー  | 並び替え内容            |
| --- | ----------------- |
| `d` | ツール起動後の CPU 使用時間順 |
| `p` | CPU 使用率順          |
| `m` | メモリ使用率順           |
| `t` | 合計 CPU 時間順        |
| `n` | PID 順             |
| `R` | 読み書き合計バイト数順       |
| `B` | 読み込みバイト数順         |
| `W` | 書き込みバイト数順         |
| `C` | キャンセルされた書き込みバイト数順 |
| `S` | 読み書きシステムコール合計順    |
| `H` | 読み書き文字数合計順        |
| `q` | 終了                |

## 更新間隔を変更する

標準では、1 秒ごとに画面が更新されます。

更新間隔を変えたい場合は、`--delay` または `-d` を使います。

```bash
python3 mytop.py --delay 2
```

この例では、2 秒ごとに表示が更新されます。

## 手動更新モード

自動更新ではなく、キーを押したときだけ更新したい場合は、`--manual-refresh` を使います。

```bash
python3 mytop.py --manual-refresh
```

このモードでは、スペースキーや Enter キーを押すと表示が更新されます。不要な更新を減らしたい場合に便利です。

## 表示する列を選ぶ

`--columns` を使うと、表示する列を指定できます。

```bash
python3 mytop.py --columns pid,user,pcpu,pmem,read_bytes,write_bytes,command
```

この例では、PID、ユーザー、CPU 使用率、メモリ使用率、読み込み量、書き込み量、コマンドだけを表示します。

利用できる列を確認したい場合は、次のように実行します。

```bash
python3 mytop.py --list-columns
```

## まとめ

Linux のプロセス状態を端末上で確認するための軽量な監視ツールです。

CPU やメモリだけでなく、プロセスごとの I/O 情報も確認できるため、システム負荷の原因調査に役立ちます。更新間隔の変更、手動更新、列の指定、キー操作による並び替えに対応しており、必要な情報を絞って確認できます。

```python
#!/usr/bin/env python3
import argparse
import curses
import os
import pwd
import signal
import time
from dataclasses import dataclass
from typing import Callable, Dict, List, Optional, Tuple


CLK_TCK = os.sysconf(os.sysconf_names["SC_CLK_TCK"])
PAGE_SIZE = os.sysconf("SC_PAGE_SIZE")

IO_FIELDS = (
    "rchar",
    "wchar",
    "syscr",
    "syscw",
    "read_bytes",
    "write_bytes",
    "cancelled_write_bytes",
)

IO_SORT_KEYS = (
    "rchar",
    "wchar",
    "rwchar",
    "syscr",
    "syscw",
    "syscrw",
    "read_bytes",
    "write_bytes",
    "rw_bytes",
    "cancelled_write_bytes",
)


@dataclass
class ProcSample:
    pid: int
    user: str
    uid: int
    cpu_ticks: int          # utime + stime
    starttime: int          # /proc/<pid>/stat field 22
    rss_bytes: int
    command: str
    io: Dict[str, int]


@dataclass
class ProcRow:
    pid: int
    user: str
    pcpu: float
    pmem: float
    dtime_ticks: int
    total_ticks: int
    command: str

    # Deltas since this tool started.
    rchar: int
    wchar: int
    syscr: int
    syscw: int
    read_bytes: int
    write_bytes: int
    cancelled_write_bytes: int

    # Derived deltas since this tool started.
    syscrw: int
    rwchar: int
    rw_bytes: int


@dataclass(frozen=True)
class ColumnSpec:
    key: str
    title: str
    width: Optional[int]   # None means variable width
    align: str             # "left" or "right"
    formatter: Callable[[ProcRow], str]


def read_mem_total_bytes() -> int:
    with open("/proc/meminfo", "r", encoding="utf-8") as f:
        for line in f:
            if line.startswith("MemTotal:"):
                # kB
                return int(line.split()[1]) * 1024
    return 1


def uid_to_user(uid: int, cache: Dict[int, str]) -> str:
    if uid in cache:
        return cache[uid]
    try:
        name = pwd.getpwuid(uid).pw_name
    except KeyError:
        name = str(uid)
    cache[uid] = name
    return name


def read_uid(pid: int) -> Optional[int]:
    try:
        with open(f"/proc/{pid}/status", "r", encoding="utf-8", errors="replace") as f:
            for line in f:
                if line.startswith("Uid:"):
                    return int(line.split()[1])  # effective uid
    except OSError:
        return None
    return None


def read_command(pid: int, fallback_comm: str) -> str:
    try:
        with open(f"/proc/{pid}/cmdline", "rb") as f:
            data = f.read()
        if data:
            parts = data.rstrip(b"\0").split(b"\0")
            cmd = " ".join(p.decode("utf-8", errors="replace") for p in parts)
            return cmd
    except OSError:
        pass
    return fallback_comm


def read_proc_io(pid: int) -> Dict[str, int]:
    """
    Read /proc/<pid>/io.

    Missing or unreadable fields are treated as 0.
    /proc/<pid>/io can be unreadable depending on permissions,
    hidepid mount options, or process lifetime races.
    """
    values = {k: 0 for k in IO_FIELDS}

    try:
        with open(f"/proc/{pid}/io", "r", encoding="utf-8", errors="replace") as f:
            for line in f:
                if ":" not in line:
                    continue

                key, value = line.split(":", 1)
                key = key.strip()

                if key not in values:
                    continue

                try:
                    values[key] = int(value.strip())
                except ValueError:
                    values[key] = 0
    except OSError:
        pass

    return values


def parse_proc_stat(pid: int) -> Optional[Tuple[str, int, int, int]]:
    """
    Return:
      comm, cpu_ticks, starttime, rss_pages

    /proc/<pid>/stat fields:
      14 utime
      15 stime
      22 starttime
      24 rss
    """
    try:
        with open(f"/proc/{pid}/stat", "r", encoding="utf-8", errors="replace") as f:
            s = f.read()
    except OSError:
        return None

    # comm is inside parentheses and may contain spaces or ')'.
    rparen = s.rfind(")")
    if rparen == -1:
        return None

    lparen = s.find("(")
    if lparen == -1:
        return None

    comm = s[lparen + 1:rparen]
    rest = s[rparen + 2:].split()

    # rest[0] is field 3: state
    try:
        utime = int(rest[11])       # field 14
        stime = int(rest[12])       # field 15
        starttime = int(rest[19])   # field 22
        rss_pages = int(rest[21])   # field 24
    except (IndexError, ValueError):
        return None

    return comm, utime + stime, starttime, rss_pages


def list_pids() -> List[int]:
    pids = []
    for name in os.listdir("/proc"):
        if name.isdigit():
            pids.append(int(name))
    return pids


def read_processes(user_cache: Dict[int, str]) -> Dict[int, ProcSample]:
    samples: Dict[int, ProcSample] = {}

    for pid in list_pids():
        parsed = parse_proc_stat(pid)
        if parsed is None:
            continue

        comm, cpu_ticks, starttime, rss_pages = parsed

        uid = read_uid(pid)
        if uid is None:
            continue

        user = uid_to_user(uid, user_cache)
        command = read_command(pid, comm)
        io = read_proc_io(pid)

        samples[pid] = ProcSample(
            pid=pid,
            user=user,
            uid=uid,
            cpu_ticks=cpu_ticks,
            starttime=starttime,
            rss_bytes=max(0, rss_pages) * PAGE_SIZE,
            command=command,
            io=io,
        )

    return samples


def format_time_plus(ticks: int) -> str:
    """
    Similar to top's TIME+.
    CPU ticks -> H:MM:SS.hh
    """
    total_seconds = ticks / CLK_TCK
    hours = int(total_seconds // 3600)
    minutes = int((total_seconds % 3600) // 60)
    seconds = int(total_seconds % 60)
    hundredths = int((total_seconds - int(total_seconds)) * 100)

    if hours > 0:
        return f"{hours}:{minutes:02d}:{seconds:02d}.{hundredths:02d}"
    return f"{minutes:02d}.{seconds:02d}"


def format_count(n: int) -> str:
    """
    Compact integer formatter for counters.
    """
    sign = "-" if n < 0 else ""
    n = abs(n)

    units = (
        (1_000_000_000_000, "T"),
        (1_000_000_000, "G"),
        (1_000_000, "M"),
        (1_000, "K"),
    )

    for base, suffix in units:
        if n >= base:
            value = n / base
            if value >= 100:
                return f"{sign}{value:.0f}{suffix}"
            if value >= 10:
                return f"{sign}{value:.1f}{suffix}"
            return f"{sign}{value:.2f}{suffix}"

    return f"{sign}{n}"


def truncate(s: str, width: int) -> str:
    if width <= 0:
        return ""
    if len(s) <= width:
        return s
    if width == 1:
        return "…"
    return s[:width - 1] + "…"


def format_cell(value: str, width: int, align: str) -> str:
    value = truncate(value, width)
    if align == "left":
        return f"{value:<{width}}"
    return f"{value:>{width}}"


def get_column_specs() -> Dict[str, ColumnSpec]:
    return {
        "pid": ColumnSpec("pid", "PID", 7, "right", lambda r: str(r.pid)),
        "user": ColumnSpec("user", "USER", 10, "left", lambda r: r.user),
        "pcpu": ColumnSpec("pcpu", "%CPU", 6, "right", lambda r: f"{r.pcpu:.1f}"),
        "pmem": ColumnSpec("pmem", "%MEM", 6, "right", lambda r: f"{r.pmem:.1f}"),
        "dtime": ColumnSpec("dtime", "DTIME+", 12, "right", lambda r: format_time_plus(r.dtime_ticks)),
        "time": ColumnSpec("time", "TIME+", 12, "right", lambda r: format_time_plus(r.total_ticks)),

        "rchar": ColumnSpec("rchar", "RCHAR", 9, "right", lambda r: format_count(r.rchar)),
        "wchar": ColumnSpec("wchar", "WCHAR", 9, "right", lambda r: format_count(r.wchar)),
        "rwchar": ColumnSpec("rwchar", "RWCHAR", 9, "right", lambda r: format_count(r.rwchar)),

        "syscr": ColumnSpec("syscr", "SYSCR", 9, "right", lambda r: format_count(r.syscr)),
        "syscw": ColumnSpec("syscw", "SYSCW", 9, "right", lambda r: format_count(r.syscw)),
        "syscrw": ColumnSpec("syscrw", "SYSCRW", 9, "right", lambda r: format_count(r.syscrw)),

        "read_bytes": ColumnSpec("read_bytes", "RBYTES", 9, "right", lambda r: format_count(r.read_bytes)),
        "write_bytes": ColumnSpec("write_bytes", "WBYTES", 9, "right", lambda r: format_count(r.write_bytes)),
        "rw_bytes": ColumnSpec("rw_bytes", "RWBYTES", 9, "right", lambda r: format_count(r.rw_bytes)),
        "cancelled_write_bytes": ColumnSpec(
            "cancelled_write_bytes",
            "CANCEL",
            9,
            "right",
            lambda r: format_count(r.cancelled_write_bytes),
        ),

        "command": ColumnSpec("command", "COMMAND", None, "left", lambda r: r.command),
    }


DEFAULT_COLUMNS = [
    "pid",
    "user",
    "pcpu",
    "pmem",
    "dtime",
    "time",
    "read_bytes",
    "write_bytes",
    "rw_bytes",
    "rchar",
    "wchar",
    "rwchar",
    "syscr",
    "syscw",
    "syscrw",
    "cancelled_write_bytes",
    "command",
]


def parse_columns(value: str, specs: Dict[str, ColumnSpec]) -> List[str]:
    columns = [c.strip() for c in value.split(",") if c.strip()]

    if not columns:
        raise argparse.ArgumentTypeError("--columns must contain at least one column")

    unknown = [c for c in columns if c not in specs]
    if unknown:
        valid = ", ".join(specs.keys())
        raise argparse.ArgumentTypeError(
            f"unknown column(s): {', '.join(unknown)}. Valid columns: {valid}"
        )

    return columns


def delta_io(cur: ProcSample, base_io: Dict[str, int]) -> Dict[str, int]:
    """
    Calculate /proc/<pid>/io deltas since this tool started.
    Counter regressions are clamped to 0.
    """
    d = {}
    for key in IO_FIELDS:
        d[key] = max(0, cur.io.get(key, 0) - base_io.get(key, 0))
    return d


def build_rows(
    current: Dict[int, ProcSample],
    previous: Dict[int, ProcSample],
    baselines: Dict[int, Tuple[int, int, Dict[str, int]]],
    interval: float,
    mem_total: int,
) -> List[ProcRow]:
    rows: List[ProcRow] = []

    for pid, cur in current.items():
        # PID reuse detection.
        # If the same PID has a different start time, it's a different process,
        # so reset the baseline.
        baseline = baselines.get(pid)
        if baseline is None or baseline[0] != cur.starttime:
            baselines[pid] = (cur.starttime, cur.cpu_ticks, dict(cur.io))
            base_ticks = cur.cpu_ticks
            base_io = cur.io
        else:
            base_ticks = baseline[1]
            base_io = baseline[2]

        dtime_ticks = max(0, cur.cpu_ticks - base_ticks)
        dio = delta_io(cur, base_io)

        prev = previous.get(pid)
        if prev is not None and prev.starttime == cur.starttime and interval > 0:
            delta_ticks = max(0, cur.cpu_ticks - prev.cpu_ticks)
            pcpu = (delta_ticks / CLK_TCK) / interval * 100.0
        else:
            pcpu = 0.0

        pmem = cur.rss_bytes / mem_total * 100.0 if mem_total > 0 else 0.0

        syscrw = dio["syscr"] + dio["syscw"]
        rwchar = dio["rchar"] + dio["wchar"]
        rw_bytes = dio["read_bytes"] + dio["write_bytes"]

        rows.append(
            ProcRow(
                pid=pid,
                user=cur.user,
                pcpu=pcpu,
                pmem=pmem,
                dtime_ticks=dtime_ticks,
                total_ticks=cur.cpu_ticks,
                command=cur.command,

                rchar=dio["rchar"],
                wchar=dio["wchar"],
                syscr=dio["syscr"],
                syscw=dio["syscw"],
                read_bytes=dio["read_bytes"],
                write_bytes=dio["write_bytes"],
                cancelled_write_bytes=dio["cancelled_write_bytes"],

                syscrw=syscrw,
                rwchar=rwchar,
                rw_bytes=rw_bytes,
            )
        )

    # Clean up baselines for terminated PIDs.
    live_pids = set(current.keys())
    for pid in list(baselines.keys()):
        if pid not in live_pids:
            baselines.pop(pid, None)

    return rows


def draw(
    stdscr,
    rows: List[ProcRow],
    refresh_sec: float,
    sort_key: str,
    start_mono: float,
    columns: List[str],
    manual_refresh: bool,
):
    stdscr.erase()
    height, width = stdscr.getmaxyx()

    specs = get_column_specs()
    selected_specs = [specs[c] for c in columns]

    elapsed = time.monotonic() - start_mono
    now = time.strftime("%H:%M:%S")

    if manual_refresh:
        refresh_label = "refresh: manual"
        refresh_hint = "Space/Enter:update "
    else:
        refresh_label = f"delay: {refresh_sec:g}s"
        refresh_hint = ""

    title = (
        f"dtop - {now}  "
        f"uptime-of-tool: {elapsed:,.1f}s  "
        f"{refresh_label}  "
        f"sort: {sort_key}  "
        f"q:quit {refresh_hint}"
        f"d:DTIME p:CPU m:MEM t:TIME n:PID "
        f"R:RWBYTES B:RBYTES W:WBYTES C:CANCEL S:SYSCRW H:RWCHAR"
    )
    stdscr.addnstr(0, 0, title, width - 1, curses.A_BOLD)

    screen_width = max(0, width - 1)
    separators_width = max(0, len(selected_specs) - 1)

    fixed_width = 0
    variable_count = 0
    for spec in selected_specs:
        if spec.width is None:
            variable_count += 1
        else:
            fixed_width += spec.width

    remaining_width = max(0, screen_width - fixed_width - separators_width)
    variable_width = remaining_width // variable_count if variable_count > 0 else 0

    def effective_width(spec: ColumnSpec) -> int:
        if spec.width is not None:
            return spec.width
        return variable_width

    header_parts = []
    for spec in selected_specs:
        w = effective_width(spec)
        header_parts.append(format_cell(spec.title, w, spec.align))

    header = " ".join(header_parts)
    stdscr.addnstr(1, 0, truncate(header, screen_width), screen_width, curses.A_REVERSE)

    max_rows = max(0, height - 2)

    for i, r in enumerate(rows[:max_rows], start=2):
        parts = []
        for spec in selected_specs:
            w = effective_width(spec)
            value = spec.formatter(r)
            parts.append(format_cell(value, w, spec.align))

        line = " ".join(parts)

        try:
            stdscr.addnstr(i, 0, truncate(line, screen_width), screen_width)
        except curses.error:
            pass

    stdscr.refresh()


def sort_rows(rows: List[ProcRow], key: str) -> List[ProcRow]:
    if key == "pid":
        return sorted(rows, key=lambda r: r.pid)
    if key == "cpu":
        return sorted(rows, key=lambda r: r.pcpu, reverse=True)
    if key == "mem":
        return sorted(rows, key=lambda r: r.pmem, reverse=True)
    if key == "time":
        return sorted(rows, key=lambda r: r.total_ticks, reverse=True)

    if key in IO_SORT_KEYS:
        return sorted(rows, key=lambda r: getattr(r, key), reverse=True)

    # Default: cumulative CPU time since this tool started.
    return sorted(rows, key=lambda r: r.dtime_ticks, reverse=True)


def key_to_sort_key(ch: int, current_sort_key: str) -> Tuple[str, bool]:
    """
    Return:
      new_sort_key, should_quit
    """
    if ch in (ord("q"), ord("Q")):
        return current_sort_key, True
    if ch in (ord("d"), ord("D")):
        return "dtime", False
    if ch in (ord("p"), ord("P")):
        return "cpu", False
    if ch in (ord("m"), ord("M")):
        return "mem", False
    if ch in (ord("t"), ord("T")):
        return "time", False
    if ch in (ord("n"), ord("N")):
        return "pid", False

    # I/O sort shortcuts.
    if ch == ord("R"):
        return "rw_bytes", False
    if ch == ord("B"):
        return "read_bytes", False
    if ch == ord("W"):
        return "write_bytes", False
    if ch == ord("C"):
        return "cancelled_write_bytes", False
    if ch == ord("S"):
        return "syscrw", False
    if ch == ord("H"):
        return "rwchar", False

    # Lowercase variants for raw counters.
    if ch == ord("r"):
        return "rchar", False
    if ch == ord("w"):
        return "wchar", False
    if ch == ord("s"):
        return "syscr", False
    if ch == ord("c"):
        return "syscw", False

    # In manual-refresh mode, Space/Enter or any other non-quit key can be used
    # as an explicit refresh trigger without changing the current sort key.
    return current_sort_key, False


def redraw_existing_rows(
    stdscr,
    args,
    rows: List[ProcRow],
    sort_key: str,
    start_mono: float,
) -> None:
    """
    Redraw already sampled rows without reading /proc again.

    This is used for curses-side repaint requests such as KEY_RESIZE.
    It must not update CPU or I/O statistics.
    """
    draw(
        stdscr=stdscr,
        rows=rows,
        refresh_sec=args.delay,
        sort_key=sort_key,
        start_mono=start_mono,
        columns=args.columns,
        manual_refresh=args.manual_refresh,
    )


def sample_and_draw(
    stdscr,
    args,
    user_cache: Dict[int, str],
    mem_total: int,
    baselines: Dict[int, Tuple[int, int, Dict[str, int]]],
    previous: Dict[int, ProcSample],
    sort_key: str,
    start_mono: float,
    last_sample_mono: float,
) -> Tuple[Dict[int, ProcSample], float, List[ProcRow]]:
    now = time.monotonic()
    current = read_processes(user_cache)
    interval = max(0.001, now - last_sample_mono)

    rows = build_rows(
        current=current,
        previous=previous,
        baselines=baselines,
        interval=interval,
        mem_total=mem_total,
    )
    rows = sort_rows(rows, sort_key)

    draw(
        stdscr=stdscr,
        rows=rows,
        refresh_sec=args.delay,
        sort_key=sort_key,
        start_mono=start_mono,
        columns=args.columns,
        manual_refresh=args.manual_refresh,
    )

    return current, now, rows


def curses_main(stdscr, args):
    curses.curs_set(0)

    if args.manual_refresh:
        stdscr.nodelay(False)
        stdscr.timeout(-1)
    else:
        stdscr.nodelay(True)
        stdscr.timeout(100)

    user_cache: Dict[int, str] = {}
    mem_total = read_mem_total_bytes()

    baselines: Dict[int, Tuple[int, int, Dict[str, int]]] = {}
    previous: Dict[int, ProcSample] = {}

    sort_key = args.sort
    start_mono = time.monotonic()
    last_sample_mono = start_mono

    # Initial sample: use this as the DTIME+ and /proc/<pid>/io baseline.
    current = read_processes(user_cache)
    for pid, p in current.items():
        baselines[pid] = (p.starttime, p.cpu_ticks, dict(p.io))
    previous = current

    # Initial draw. Deltas are zero at startup.
    rows = build_rows(
        current=current,
        previous=previous,
        baselines=baselines,
        interval=0.001,
        mem_total=mem_total,
    )
    rows = sort_rows(rows, sort_key)
    draw(
        stdscr=stdscr,
        rows=rows,
        refresh_sec=args.delay,
        sort_key=sort_key,
        start_mono=start_mono,
        columns=args.columns,
        manual_refresh=args.manual_refresh,
    )

    next_sample = time.monotonic() + args.delay

    while True:
        if args.manual_refresh:
            ch = stdscr.getch()

            if ch == curses.KEY_RESIZE:
                # Terminal resize / curses repaint request:
                # redraw using already sampled rows only.
                redraw_existing_rows(
                    stdscr=stdscr,
                    args=args,
                    rows=rows,
                    sort_key=sort_key,
                    start_mono=start_mono,
                )
                continue

            sort_key, should_quit = key_to_sort_key(ch, sort_key)
            if should_quit:
                break

            previous, last_sample_mono, rows = sample_and_draw(
                stdscr=stdscr,
                args=args,
                user_cache=user_cache,
                mem_total=mem_total,
                baselines=baselines,
                previous=previous,
                sort_key=sort_key,
                start_mono=start_mono,
                last_sample_mono=last_sample_mono,
            )
            continue

        ch = stdscr.getch()
        if ch != -1:
            if ch == curses.KEY_RESIZE:
                # Terminal resize / curses repaint request:
                # redraw using already sampled rows only. Do not reset
                # next_sample and do not read /proc here.
                redraw_existing_rows(
                    stdscr=stdscr,
                    args=args,
                    rows=rows,
                    sort_key=sort_key,
                    start_mono=start_mono,
                )
                continue

            sort_key, should_quit = key_to_sort_key(ch, sort_key)
            if should_quit:
                break

            # In auto mode, a normal key-driven sort/refresh should redraw
            # immediately using a fresh sample instead of waiting for the next
            # interval. curses.KEY_RESIZE is handled above and is redraw-only.
            previous, last_sample_mono, rows = sample_and_draw(
                stdscr=stdscr,
                args=args,
                user_cache=user_cache,
                mem_total=mem_total,
                baselines=baselines,
                previous=previous,
                sort_key=sort_key,
                start_mono=start_mono,
                last_sample_mono=last_sample_mono,
            )
            next_sample = last_sample_mono + args.delay

        now = time.monotonic()
        if now >= next_sample:
            previous, last_sample_mono, rows = sample_and_draw(
                stdscr=stdscr,
                args=args,
                user_cache=user_cache,
                mem_total=mem_total,
                baselines=baselines,
                previous=previous,
                sort_key=sort_key,
                start_mono=start_mono,
                last_sample_mono=last_sample_mono,
            )
            next_sample = last_sample_mono + args.delay

        time.sleep(0.02)


def parse_args():
    specs = get_column_specs()

    sort_choices = [
        "dtime",
        "cpu",
        "mem",
        "time",
        "pid",
        *IO_SORT_KEYS,
    ]

    parser = argparse.ArgumentParser(
        description=(
            "top-like process viewer with DTIME+ and /proc/<pid>/io deltas "
            "since this tool started; optional manual refresh mode"
        )
    )
    parser.add_argument(
        "-d",
        "--delay",
        type=float,
        default=1.0,
        help="refresh interval seconds, default: 1.0",
    )
    parser.add_argument(
        "--manual-refresh",
        action="store_true",
        help=(
            "disable periodic updates and refresh only when a key is pressed. "
            "Space/Enter refresh without changing the current sort key."
        ),
    )
    parser.add_argument(
        "-s",
        "--sort",
        choices=sort_choices,
        default="dtime",
        help="initial sort key, default: dtime",
    )
    parser.add_argument(
        "--columns",
        type=lambda s: parse_columns(s, specs),
        default=DEFAULT_COLUMNS,
        help=(
            "comma-separated columns to display. "
            "Use --list-columns to show available columns."
        ),
    )
    parser.add_argument(
        "--list-columns",
        action="store_true",
        help="list available column keys and exit",
    )

    args = parser.parse_args()

    if args.list_columns:
        for key, spec in specs.items():
            print(f"{key:<24} {spec.title}")
        raise SystemExit(0)

    return args


def main():
    signal.signal(signal.SIGINT, signal.SIG_DFL)
    args = parse_args()
    curses.wrapper(curses_main, args)


if __name__ == "__main__":
    main()
```
