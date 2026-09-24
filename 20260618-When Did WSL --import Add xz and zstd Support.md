---
pubDatetime: 2026-06-18T16:49:37+09:00
title: "WSL の --import はいつ xz zstd にも対応したのか"
description: ".tar.xz は少なくとも WSL 1.1.3.0 で動いていた、.tar.zst は少なくとも WSL 2.1.5 で動いていた、2020年時点では少なくとも明示対応ではなかったを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

結論としては、**「このバージョンから正式対応」と言える公式リリースノートは見つかりません**。ただし公開情報から確認できる下限はあります。

| 形式         |         公開情報から確認できる最古付近の動作確認 | 確度  |
| ---------- | ---------------------------: | --- |
| `.tar.xz`  | **WSL 1.1.3.0、2023-03-12時点** | 高め  |
| `.tar.zst` |   **WSL 2.1.5、2024-04-11時点** | 中程度 |

## `.tar.xz` は少なくとも WSL 1.1.3.0 で動いていた

GitHub issue #9778 では、報告環境が **WSL Version 1.1.3.0** で、再現手順に `wsl --import ... lunar-server-cloudimg-amd64-root.tar.xz` が含まれています。問題は import 失敗ではなく、その後 VS Code から開いた際の不具合なので、少なくともその時点で `.tar.xz` を `wsl --import` に渡す運用は成立していたと読めます。([GitHub][1])

## `.tar.zst` は少なくとも WSL 2.1.5 で動いていた

GitHub issue #6056 のコメントとして、**2024-04-11** に「latest 2.1.5 で `.tar.gz`, `.tar.xz`, `.tar.zst` の import が動く」との確認報告が出ています。これは公式リリースノートではなくコミュニティ報告ですが、`.tar.zst` について見つかった確認情報としては有用です。([GitHub][2])

## 2020年時点では少なくとも明示対応ではなかった

2020年10月の issue #6056 は、`wsl.exe --import` / WSL Distro Launcher 系で `.xz` を使いたい、という要望として立てられています。本文では当時の想定が `install.tar` または gzip 圧縮の `install.tar.gz` / `install.tgz` であることに触れたうえで、XZ 対応を求めています。つまり、**2020年時点では `.tar.xz` は少なくとも明示的・公式に前提化された形式ではなかった**と見てよいです。([GitHub][3])

## 実装上は「import 側で拡張子を厳密に列挙している」わけではなさそう

現在の WSL ソースを見ると、`wsl --import` の処理は、通常 import の場合に `.vhd` / `.vhdx` を `--vhd` なしで渡したケースを弾き、ファイルを開いて `RegisterDistribution` に渡す構造です。`wsl.exe` 側で `.gz` / `.xz` / `.zst` を明示的に分岐しているようには見えません。([GitHub][4])

一方、`--export` 側には `--format tar.gz`, `tar.xz`, `vhd`, `tar` のような明示的な形式指定があります。つまり **export は形式が列挙されているが、import は tar 系アーカイブとして渡してバックエンド側の展開処理に任せている**、という理解が近いです。([GitHub][4])

## まとめ

厳密にはこうです。

> `.tar.xz` は **遅くとも WSL 1.1.3.0、2023年3月時点**で `wsl --import` に使われていた。
> `.tar.zst` は **遅くとも WSL 2.1.5、2024年4月時点**で動作報告がある。
> ただし、どちらも「このリリースで対応開始」と明記した公式 changelog は確認できない。

したがって、**WSL 2.5.x で初対応したわけではありません**。特に `.tar.xz` はそれよりかなり前から動いていたと見てよいです。

[1]: https://github.com/microsoft/WSL/issues/9778 "Problem running code and code-insiders on Ubuntu distro 23.04 · Issue #9778 · microsoft/WSL · GitHub"
[2]: https://github.com/microsoft/WSL/issues/6056?utm_source=chatgpt.com "Add .xz archive support to wsl.exe --import to improve ..."
[3]: https://github.com/microsoft/WSL/issues/6056?timeline_page=1 "Add .xz archive support to wsl.exe --import to improve microsoft/WSL-DistroLauncher · Issue #6056 · microsoft/WSL · GitHub"
[4]: https://github.com/microsoft/WSL/blob/master/src/windows/common/WslClient.cpp "WSL/src/windows/common/WslClient.cpp at master · microsoft/WSL · GitHub"
