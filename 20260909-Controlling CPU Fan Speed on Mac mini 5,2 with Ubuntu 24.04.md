---
title: "macmini5,2 ubuntu24.04でcpuファンスピードをコントロールしたい"
description: "まず applesmc を確認、手動で回転数を変更する、おすすめ：mbpfan で温度連動を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-09-09T01:54:03.910Z
---

Macmini5,2（Mid 2011）+ Ubuntu 24.04 なら、**`applesmc` + `mbpfan`** が扱いやすい構成です。Ubuntu 24.04（Noble）にも `mbpfan 2.4.0` が標準パッケージとしてあります。([Ubuntu Packages][1])

Linux の `applesmc` ドライバは Intel Mac の温度センサーとファン制御を sysfs 経由で提供しています。([GitHub][2])

## 1. まず `applesmc` を確認

```bash
sudo modprobe applesmc
sudo modprobe coretemp

lsmod | grep -E 'applesmc|coretemp'
```

次にファンのインターフェースを確認します。

```bash
SMC=$(find /sys/devices/platform -maxdepth 1 -type d -name 'applesmc.*' -print -quit)
echo "$SMC"

ls -l "$SMC"/fan*
```

例えば、以下のようなものが出れば制御できます。

```text
fan1_input
fan1_manual
fan1_max
fan1_min
fan1_output
```

`fan1_min` と `fan1_max` を必ず確認してください。

```bash
cat "$SMC/fan1_min"
cat "$SMC/fan1_max"
cat "$SMC/fan1_input"
```

Mac によって値は違うので、**1800～5500 RPM などと決め打ちしない方が安全**です。

---

## 2. 手動で回転数を変更する

まず一時的に 3000 RPM などでテストできます。

```bash
echo 1 | sudo tee "$SMC/fan1_manual"
echo 3000 | sudo tee "$SMC/fan1_output"
```

現在の実回転数：

```bash
cat "$SMC/fan1_input"
```

数秒すると 3000 RPM 前後になっているはずです。

Linux 上の Intel Mac では、`fan1_manual=1` にしてから `fan1_output` に目標RPMを書く方法が使われています。([Ask Ubuntu][3])

### 自動制御に戻す

これは重要です。

```bash
echo 0 | sudo tee "$SMC/fan1_manual"
```

手動モードのまま低回転に固定すると、CPU/GPU負荷が上がっても回転数が上がらなくなります。

---

## 3. おすすめ：`mbpfan` で温度連動

Ubuntu 24.04 なら、

```bash
sudo apt update
sudo apt install mbpfan lm-sensors
```

有効化：

```bash
sudo systemctl enable --now mbpfan
```

状態確認：

```bash
systemctl status mbpfan
```

`mbpfan` は `coretemp` からCPU温度を読み、`applesmc` 経由でファン速度を変更します。([GitHub][4])

設定は：

```bash
sudo nano /etc/mbpfan.conf
```

例えば私は Macmini5,2 なら、まず穏当な設定として次のあたりから試します。

```ini
[general]

low_temp = 55
high_temp = 65
max_temp = 80

polling_interval = 1
```

ファンの min/max は `applesmc` が報告する値を `mbpfan` が利用できます。明示的に設定したい場合は、

```bash
cat "$SMC/fan1_min"
cat "$SMC/fan1_max"
```

を確認してから、

```ini
min_fan1_speed = 2000
max_fan1_speed = 5500
```

のように**実機の値を使って**設定します。`mbpfan` の標準設定も `/sys/devices/platform/applesmc.768/fan*_min` と `fan*_max` を参照する仕様です。([GitHub][5])

変更後：

```bash
sudo systemctl restart mbpfan
```

---

## 4. 温度とRPMをリアルタイム監視

```bash
sensors
```

または、

```bash
watch -n 1 sensors
```

ファンRPMも直接見たいなら：

```bash
watch -n 1 "cat $SMC/fan1_input; sensors"
```

## Macmini5,2 なら私ならこうします

**常用は `mbpfan`、手動の `fan1_output` は動作確認用**にします。

最初に、次の結果を貼ってもらえれば、Macmini5,2 の実機値に合わせて `/etc/mbpfan.conf` を具体的に作れます。

```bash
uname -r
lsmod | grep -E 'applesmc|coretemp'
SMC=$(find /sys/devices/platform -maxdepth 1 -type d -name 'applesmc.*' -print -quit)
echo "$SMC"
for f in "$SMC"/fan*; do echo -n "$f = "; cat "$f"; done
sensors
```

特に **`fan1_min` / `fan1_max` とアイドル時・負荷時CPU温度**が分かれば、「静音寄り」「冷却寄り」のファンカーブを決められます。

[1]: https://packages.ubuntu.com/ja/source/noble/mbpfan?utm_source=chatgpt.com "Ubuntu – noble の mbpfan ソースパッケージに関する詳細"
[2]: https://github.com/torvalds/linux/blob/master/drivers/hwmon/Kconfig?utm_source=chatgpt.com "linux/drivers/hwmon/Kconfig at master · torvalds/linux · GitHub"
[3]: https://askubuntu.com/questions/1413150/macmini-2010-with-ubuntu-22-04-temperature-problems-fan-control?utm_source=chatgpt.com "MacMini 2010 with Ubuntu 22.04 - temperature problems (Fan Control) - Ask Ubuntu"
[4]: https://github.com/linux-on-mac/mbpfan?utm_source=chatgpt.com "GitHub - linux-on-mac/mbpfan: A simple daemon to control fan speed on all MacBook/MacBook Pros (probably all Apple computers) for Linux Kernel 3 and newer · GitHub"
[5]: https://github.com/linux-on-mac/mbpfan/blob/master/mbpfan.conf?utm_source=chatgpt.com "mbpfan/mbpfan.conf at master · linux-on-mac/mbpfan · GitHub"
