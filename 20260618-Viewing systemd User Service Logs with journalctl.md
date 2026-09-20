---
pubDatetime: 2026-06-18T13:43:56+09:00
title: "systemd user service のログを journalctl で確認する"
description: "`systemctl --user status hoge` で確認している user service のログは、`journalctl` では `--user-unit` を使って確認できる。"
---

`systemctl --user status hoge` で確認している user service のログは、`journalctl` では `--user-unit` を使って確認できる。

```bash
journalctl --user-unit hoge.service
```

直近のログだけ見るなら次のようにする。

```bash
journalctl --user-unit hoge.service -n 100 --no-pager
```

リアルタイムで追跡したい場合は `-f` を付ける。

```bash
journalctl --user-unit hoge.service -f
```

今回のブート以降のログに絞る場合は `-b` を使う。

```bash
journalctl --user-unit hoge.service -b
```

時刻付きで見やすくするなら、次の指定が便利。

```bash
journalctl --user-unit hoge.service -n 100 --no-pager --output=short-iso
```

エラーや警告だけ確認したい場合は、priority を指定する。

```bash
journalctl --user-unit hoge.service -p warning..alert
```

まとめると、まずは次のコマンドを使えばよい。

```bash
journalctl --user-unit hoge.service -n 100 --no-pager
```

`systemctl --user status hoge` に対応するログ確認では、`journalctl -u hoge` ではなく `journalctl --user-unit hoge.service` を使う。
