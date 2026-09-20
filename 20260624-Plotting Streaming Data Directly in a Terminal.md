---
pubDatetime: 2026-06-24T17:37:47+09:00
title: "ターミナル上でストリームデータを直接プロットする"
description: "このツールは、ライブの数値データをターミナル内に直接プロットするための小さなコマンドラインプログラムです。標準入力から値を読み取り、それらを gnuplot に渡し、新しいデータが到着するたびに更新されるシンプルなテキストベースのグラフを表示します。 目的 このツールの目的は、別のグラフィカルウィン…"
---

このツールは、ライブの数値データをターミナル内に直接プロットするための小さなコマンドラインプログラムです。標準入力から値を読み取り、それらを `gnuplot` に渡し、新しいデータが到着するたびに更新されるシンプルなテキストベースのグラフを表示します。

## 目的

このツールの目的は、別のグラフィカルウィンドウを開くことなく、ストリームデータを簡単に観察できるようにすることです。

センサーの測定値、ログ出力、ベンチマーク結果、または他のコマンドラインプログラムが生成する値など、リアルタイムで変化するデータを監視したい場合に役立ちます。

グラフはターミナル内に描画されるため、シンプルなシェルワークフローや、グラフィカルデスクトップ環境が利用できないリモート環境でも使用できます。

## 動作の仕組み

プログラムは標準入力から行単位でデータを読み取ります。各行には数値データが含まれていることを想定しています。

有効な数値を受け取るたびに、プログラムは起動からの経過時間を記録し、その値を経過時間に対してプロットします。新しいデータを受信すると、グラフは自動的に更新されます。

保持されるデータポイント数には上限があります。デフォルトでは最大 200 点まで保存されるため、表示は直近の変化に集中できます。

## 入力形式

このツールはカンマ区切りまたはタブ区切りの入力を受け付けます。

例えば、以下の入力をプロットできます。

```text
1.2
1.5
1.8
2.1
```

また、CSV 形式のようなデータから最初の列を読み取ることもできます。

```text
1.2,ok
1.5,ok
1.8,ok
2.1,ok
```

行に有効な数値が含まれていない場合、その行は無視されます。

## 基本的な使い方

スクリプトをファイルとして保存します。例えば次のようにします。

```bash
stream-plot.py
```

実行可能にします。

```bash
chmod +x stream-plot.py
```

その後、数値データをパイプで渡します。

```bash
python3 generate_values.py | ./stream-plot.py
```

ターミナル内にシンプルなライブプロットが表示されます。

## 使用例

シェルのループを使って簡単にテストできます。

```bash
while true; do
  echo $RANDOM
  sleep 1
done | ./stream-plot.py
```

この例では 1 秒ごとにランダムな値が生成されます。新しい値が入力されるたびに、ターミナル上のプロットが更新されます。

## ターミナルサイズの変更

このツールはターミナルのサイズを検出し、プロット領域を自動調整します。ターミナルウィンドウのサイズが変更された場合でも、グラフが利用可能な表示領域に収まるようにプロットサイズが更新されます。

## ツールの終了

ツールは以下で停止できます。

```bash
Ctrl+C
```

終了時には `gnuplot` プロセスを適切にクリーンアップして閉じます。

## 必要環境

このツールには以下が必要です。

```bash
gnuplot
```

また、Python 3 と、`asyncio`、`csv`、`signal` などの標準 Python ライブラリを使用します。

## まとめ

このターミナル向けストリームプロットツールは、コマンドラインからライブの数値データを監視するためのシンプルな方法を提供します。軽量で標準入力を利用でき、フル機能のプロットアプリケーションが不要な簡易モニタリング用途に適しています。

```python
import asyncio
import csv
import io
import shutil
import signal
import sys
import time
import uuid
from collections import deque


def get_plot_size():
    size = shutil.get_terminal_size(fallback=(120, 30))

    width = max(20, size.columns)
    height = max(10, size.lines - 1)

    return width, height


class StreamGnuplot:
    def __init__(
        self,
        *,
        max_points=200,
        width=120,
        height=30,
        value_column=0,
        # title="stdin stream",
        # ylabel="value",
    ):
        self.max_points = max_points
        self.width = width
        self.height = height
        self.value_column = value_column
        # self.title = title
        # self.ylabel = ylabel

        self.start_time = time.monotonic()
        self.points = deque(maxlen=max_points)

        self.proc = None
        self._lock = asyncio.Lock()

        self._first_draw = True
        self._resize_requested = False

    async def start(self):
        self.proc = await asyncio.create_subprocess_exec(
            "gnuplot",
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )

        await self._write(self._gnuplot_setup_script())

    def request_resize(self):
        self._resize_requested = True

    def _gnuplot_setup_script(self):

        # set title "{self.title}"
        # set ylabel "{self.ylabel}"
        return f"""
set terminal dumb {self.width} {self.height}
set print "-"
set xlabel "time since start [s]"
set grid
set key off
"""

    async def _apply_resize_if_needed(self):
        if not self._resize_requested:
            return

        self._resize_requested = False

        new_width, new_height = get_plot_size()

        if new_width == self.width and new_height == self.height:
            return

        self.width = new_width
        self.height = new_height

        await self._write(f"set terminal dumb {self.width} {self.height}\n")

        # ここでは _first_draw を戻さない。
        # 「1回目だけ \033[H しない」という条件を維持する。

    async def stop(self):
        if self.proc is None:
            return

        try:
            await self._write("exit\n")
        except Exception:
            pass

        try:
            await asyncio.wait_for(self.proc.wait(), timeout=1.0)
        except asyncio.TimeoutError:
            self.proc.terminate()
            await self.proc.wait()

    async def _write(self, text):
        if self.proc is None or self.proc.stdin is None:
            raise RuntimeError("gnuplot is not running")

        self.proc.stdin.write(text.encode("utf-8"))
        await self.proc.stdin.drain()

    async def add_value(self, value):
        elapsed = time.monotonic() - self.start_time
        self.points.append((elapsed, value))
        await self.replot()

    async def replot(self):
        if not self.points:
            return

        async with self._lock:
            await self._apply_resize_if_needed()

            marker = f"__GPLOT_DONE_{uuid.uuid4().hex}__"

            data = "\n".join(
                f"{x:.6f} {y:.12g}"
                for x, y in self.points
            )

            script = f"""
plot "-" using 1:2 with lines
{data}
e
print "{marker}"
"""

            await self._write(script)

            plot_text = await self._read_until_marker(marker)
            self._draw_to_terminal(plot_text)

    async def _read_until_marker(self, marker):
        if self.proc is None or self.proc.stdout is None:
            raise RuntimeError("gnuplot stdout is not available")

        lines = []

        while True:
            raw = await self.proc.stdout.readline()

            if raw == b"":
                raise RuntimeError("gnuplot stdout closed unexpectedly")

            line = raw.decode("utf-8", errors="replace")

            if line.rstrip("\r\n") == marker:
                break

            lines.append(line)

        return "".join(lines)

    def _draw_to_terminal(self, plot_text):
        """
        初回は cursor home せず、その場にプロットを出力する。
        2回目以降は左上へ戻ってプロットテキストを上書きする。

        プロットテキスト最後の改行だけを出力しない。
        sys.stdout.write("\\033[J") は使わない。
        """
        if self._first_draw:
            self._first_draw = False
        else:
            sys.stdout.write("\033[H")

        if plot_text.endswith("\r\n"):
            plot_text = plot_text[:-2]
        elif plot_text.endswith("\n") or plot_text.endswith("\r"):
            plot_text = plot_text[:-1]

        sys.stdout.write(plot_text)
        sys.stdout.flush()


def detect_delimiter(line):
    if "\t" in line:
        return "\t"
    return ","


def parse_numeric_value(line, value_column):
    line = line.strip()

    if not line:
        return None

    delimiter = detect_delimiter(line)

    reader = csv.reader(io.StringIO(line), delimiter=delimiter)
    row = next(reader, None)

    if not row:
        return None

    try:
        return float(row[value_column])
    except (ValueError, IndexError):
        return None


async def read_stdin_stream(plotter):
    loop = asyncio.get_running_loop()

    while True:
        line = await loop.run_in_executor(None, sys.stdin.readline)

        if line == "":
            break

        value = parse_numeric_value(line, plotter.value_column)

        if value is None:
            continue

        await plotter.add_value(value)


async def drain_stderr(proc):
    if proc.stderr is None:
        return

    while True:
        raw = await proc.stderr.readline()

        if raw == b"":
            break

        # sys.stderr.write(raw.decode("utf-8", errors="replace"))
        # sys.stderr.flush()


async def main():
    width, height = get_plot_size()

    plotter = StreamGnuplot(
        max_points=200,
        width=width,
        height=height,
        value_column=0,
        # title="stdin stream",
        # ylabel="value",
    )

    stop_event = asyncio.Event()

    def request_stop():
        stop_event.set()

    loop = asyncio.get_running_loop()

    for sig in (signal.SIGINT, signal.SIGTERM):
        try:
            loop.add_signal_handler(sig, request_stop)
        except NotImplementedError:
            pass

    try:
        loop.add_signal_handler(signal.SIGWINCH, plotter.request_resize)
    except (AttributeError, NotImplementedError):
        pass

    await plotter.start()

    stderr_task = asyncio.create_task(drain_stderr(plotter.proc))
    reader_task = asyncio.create_task(read_stdin_stream(plotter))
    stop_task = asyncio.create_task(stop_event.wait())

    done, pending = await asyncio.wait(
        {reader_task, stop_task},
        return_when=asyncio.FIRST_COMPLETED,
    )

    for task in pending:
        task.cancel()

    stderr_task.cancel()

    await plotter.stop()


if __name__ == "__main__":
    asyncio.run(main())
```
