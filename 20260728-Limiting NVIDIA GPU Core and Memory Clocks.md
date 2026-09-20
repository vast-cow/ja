---
pubDatetime: 2026-07-28T14:57:05+09:00
title: "NVIDIA GPUのコアクロック・メモリクロックを制限する方法"
description: "Linux：nvidia-smiで制限するについて、具体的な手順と注意点をまとめます。"
---

## Linux：`nvidia-smi`で制限する

対応GPUなら、コアクロックとメモリクロックの範囲を直接指定できます。`root`権限が必要です。GPUクロックのロックはVolta世代以降でサポートされていますが、製品・ドライバによっては `Not Supported` になります。([NVIDIA Docs][1])

### 1. GPUと対応クロックを確認

```bash
nvidia-smi -L
sudo nvidia-smi -q -d SUPPORTED_CLOCKS
sudo nvidia-smi -lmi
```

### 2. コアクロックを制限

GPU 0を、最低300MHz～最大1500MHzに制限する例です。

```bash
sudo nvidia-smi -i 0 --lock-gpu-clocks=300,1500
```

1500MHzに固定する場合：

```bash
sudo nvidia-smi -i 0 --lock-gpu-clocks=1500
```

単一値を指定すると固定動作になるため、アイドル時もクロックが下がりにくくなります。通常は `最低値,最大値` の範囲指定が適しています。([NVIDIA Docs][1])

### 3. メモリクロックを制限

最低810MHz～最大5000MHzの例：

```bash
sudo nvidia-smi -i 0 --lock-memory-clocks=810,5000
```

5000MHz固定：

```bash
sudo nvidia-smi -i 0 --lock-memory-clocks=5000
```

メモリクロック制御はGPUによって非対応です。また、Hopper系GPUでは通常の `--lock-memory-clocks` ではなく、遅延適用方式が必要です。([NVIDIA Docs][1])

### 4. 状態を監視

```bash
watch -n 1 'nvidia-smi --query-gpu=index,name,clocks.gr,clocks.mem,power.draw,temperature.gpu --format=csv'
```

### 5. 元に戻す

```bash
sudo nvidia-smi -i 0 --reset-gpu-clocks
sudo nvidia-smi -i 0 --reset-memory-clocks
```

### クロック制御が非対応の場合

消費電力を下げることで、間接的にブーストクロックを制限できます。

```bash
nvidia-smi -q -d POWER
sudo nvidia-smi -i 0 --power-limit=150
```

指定値は表示される `Min Power Limit`～`Max Power Limit` の範囲内にする必要があります。([NVIDIA Docs][1])


[1]: https://docs.nvidia.com/deploy/nvidia-smi/index.html "docs.nvidia.com"
[2]: https://jp.msi.com/support/technical_details/VGA_MSI_Utility_AfterBurner "〖グラフィックスカード〗MSI Afterburnerの使い方"
[3]: https://jp.msi.com/blog/msi-afterburner-overclocking-undervolting-guide?utm_source=chatgpt.com "MSI Afterburner Walkthrough Part 1: Overclocking Guide & ..."
