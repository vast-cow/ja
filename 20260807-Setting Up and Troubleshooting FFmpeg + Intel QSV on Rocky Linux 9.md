---
title: "Rocky Linux 9 で FFmpeg + Intel QSV を使うためのセットアップとトラブルシューティング"
description: ""
pubDatetime: 2026-08-07T08:31:28.851Z
---

Intel CPU の内蔵 GPU を使って FFmpeg の H.264 / HEVC エンコードを高速化したい場合、Intel Quick Sync Video（QSV）が利用できます。

しかし Linux、特に Rocky Linux 9 のような RHEL 系ディストリビューションでは、

```text
FFmpeg が h264_qsv を認識している
```

だけでは QSV は動きません。

実際、今回 Rocky Linux 9 上で次のようなエラーに遭遇しました。

```text
[AVHWDeviceContext @ 0x562042595380] Failed to initialise VAAPI connection: -1 (unknown libva error).
[h264_qsv @ 0x56204258e140] Failed to create a VAAPI device.
Error initializing output stream 0:0
```

さらに `-qsv_device` で Intel GPU を明示しても、

```text
[AVHWDeviceContext @ 0x55ac9a494500] Failed to initialise VAAPI connection: -1 (unknown libva error).
Device creation failed: -5.
Failed to set value '/dev/dri/renderD128' for option 'qsv_device': Input/output error
```

となりました。

この記事では、Rocky Linux 9 で FFmpeg + QSV をセットアップする方法と、こうしたエラーをどの順番で切り分けるべきかをまとめます。

---

## QSV が動くまでのレイヤを理解する

最初に重要なのは、QSV は FFmpeg 単体の機能ではないということです。

Linux では概念的に次のような複数のレイヤを通って Intel GPU にアクセスします。

```text
FFmpeg
  ↓
QSV
  ↓
Intel Media SDK / oneVPL
  ↓
Intel Media Driver
  ↓
VA-API / libva
  ↓
/dev/dri/renderD128
  ↓
i915 / xe
  ↓
Intel GPU
```

そのため、

```bash
ffmpeg -encoders | grep qsv
```

で

```text
h264_qsv
hevc_qsv
```

が表示されても、それだけでは GPU が実際に使えることを意味しません。

例えば、

- Intel GPU を Linux が認識していない
- `nomodeset` が設定されている
- `i915` / `xe` がロードされていない
- `/dev/dri/renderD128` にアクセスできない
- `libva` がない
- Intel Media Driver がない、またはロードできない
- Media SDK / oneVPL runtime が合っていない
- FFmpeg 自体が QSV 対応でビルドされていない

といったどこか一箇所でも問題があると、QSV は動きません。

したがって、下から順番に確認していくのがトラブルシューティングの基本です。

---

# 1. Intel GPU を Linux が認識しているか確認する

まず PCI デバイスを確認します。

```bash
lspci -nn | grep -Ei 'VGA|Display'
```

今回の環境では、

```text
00:02.0 VGA compatible controller [0300]:
Intel Corporation GeminiLake [UHD Graphics 605] [8086:3184] (rev 03)
```

となりました。

Intel UHD Graphics 605、つまり Gemini Lake の GPU が認識されています。

次に、カーネルドライバを確認します。

```bash
lspci -nnk | grep -A4 -Ei 'VGA|Display'
```

今回の結果は、

```text
00:02.0 VGA compatible controller [0300]: Intel Corporation GeminiLake [UHD Graphics 605] [8086:3184] (rev 03)
        DeviceName: Onboard - Video
        Subsystem: Elitegroup Computer Systems Device [1019:a94d]
        Kernel driver in use: i915
        Kernel modules: i915
```

でした。

ここで重要なのは、

```text
Kernel driver in use: i915
```

です。

Gemini Lake では `i915` が使われます。

最近の Intel GPU では構成によって `xe` が使われることもあります。

念のため、

```bash
lsmod | grep -E 'i915|xe'
```

でも確認できます。

---

# 2. `nomodeset` を設定している場合は外す

Intel iGPU を QSV / VA-API で使用する場合、カーネルの起動オプションに `nomodeset` を指定していると、GPU ドライバが正常に初期化されず、QSV が使えない原因になることがあります。

まず現在のカーネルコマンドラインを確認します。

```bash
cat /proc/cmdline
```

ここに、

```text
nomodeset
```

が含まれている場合は削除します。

`nomodeset` は Kernel Mode Setting（KMS）を無効化するオプションです。Intel GPU で使用する `i915` などの DRM/KMS ドライバの正常な初期化を妨げるため、`/dev/dri/renderD128` が生成されない、あるいは GPU デバイスが存在していても VA-API の初期化に失敗する原因になり得ます。

Rocky Linux 9 では GRUB の設定を確認します。

```bash
sudo grubby --info=ALL | grep args
```

`nomodeset` が設定されている場合は、全カーネルエントリから削除できます。

```bash
sudo grubby --update-kernel=ALL --remove-args="nomodeset"
```

設定後、再起動します。

```bash
sudo reboot
```

再起動後、もう一度確認します。

```bash
cat /proc/cmdline
```

`nomodeset` が消えていることを確認した上で、

```bash
lspci -nnk | grep -A4 -Ei 'VGA|Display'
```

を実行し、

```text
Kernel driver in use: i915
```

となっていることを確認します。

続いて DRM デバイスも確認します。

```bash
ls -l /dev/dri/
```

最低限、

```text
card0
renderD128
```

などが生成されていることを確認します。

---

# 3. `/dev/dri/renderD128` を確認する

次に DRM デバイスを確認します。

```bash
ls -l /dev/dri/
```

今回の環境では、

```text
drwxr-xr-x. 2 root root         80 Aug  7 16:04 by-path
crw-rw----. 1 root video  226,   0 Aug  7 16:04 card0
crw-rw-rw-. 1 root render 226, 128 Aug  7 16:04 renderD128
```

となっていました。

QSV や VA-API のサーバ用途で特に重要なのが、

```text
/dev/dri/renderD128
```

です。

X11 や Wayland を起動していないサーバでも、`renderD128` にアクセスできればハードウェアエンコードできます。

つまり、

```text
GUI がない = QSV が使えない
```

ではありません。

ヘッドレスサーバでも利用可能です。

---

# 4. renderD128 の権限を確認する

典型的には次のようになっています。

```text
crw-rw---- 1 root render ... /dev/dri/renderD128
```

この場合、FFmpeg を実行するユーザーを `render` グループへ追加します。

```bash
sudo usermod -aG render $USER
```

環境によっては `video` も必要になるため、

```bash
sudo usermod -aG video,render $USER
```

としても構いません。

再ログイン後、

```bash
id
```

で確認します。

systemd サービスから FFmpeg を起動する場合には、ログインユーザーではなく**サービスを実行しているユーザー**に権限が必要です。

Jellyfin、Plex、MediaMTX と組み合わせた独自トランスコード処理などでも、この点には注意が必要です。

今回の環境では、

```text
crw-rw-rw-. 1 root render ... renderD128
```

だったため、単純な Unix パーミッション不足である可能性は低い状態でした。

---

# 5. EPEL と RPM Fusion を有効にする

Rocky Linux 9 の標準リポジトリだけでは FFmpeg や Intel Media Driver 周辺のパッケージが不足する場合があります。

まず EPEL を追加します。

```bash
sudo dnf install -y epel-release
```

続いて RPM Fusion Free / Nonfree を追加します。

```bash
sudo dnf install -y \
  https://download1.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm \
  https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm
```

その後更新しておきます。

```bash
sudo dnf update -y
```

---

# 6. Intel Media Driver をインストールする

Intel の比較的新しい GPU では、VA-API ドライバとして Intel Media Driver、つまり `iHD` ドライバを使用します。

Rocky Linux 9 + RPM Fusion なら、

```bash
sudo dnf install -y intel-media-driver
```

をインストールします。

Intel Media Driver は VA-API バックエンドとなる、

```text
/usr/lib64/dri/iHD_drv_video.so
```

を提供します。

今回使用した Intel UHD Graphics 605 / Gemini Lake も Intel Media Driver の対象です。

---

# 7. libva と vainfo をインストールする

VA-API の動作確認には `vainfo` が非常に便利です。

```bash
sudo dnf install -y libva libva-utils
```

インストール後、

```bash
vainfo
```

を実行します。

ただし GUI のないサーバでは、DRM デバイスを明示したほうが確実です。

```bash
vainfo --display drm --device /dev/dri/renderD128
```

正常なら、概ね次のようになります。

```text
libva info: VA-API version ...
libva info: Trying to open /usr/lib64/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_...
libva info: va_openDriver() returns 0

vainfo: Driver version: Intel iHD driver ...
```

---

# 8. `LIBVA_DRIVER_NAME=iHD` で明示する

自動判定がうまくいかない場合には、使用する VA-API ドライバを明示できます。

```bash
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

恒久的に指定したければ、

```bash
export LIBVA_DRIVER_NAME=iHD
```

としても構いません。

---

# 9. Intel Media SDK と oneVPL の違い

QSV 周りでは、

```text
Intel Media SDK
libmfx
oneVPL
libvpl
intel-vpl-gpu-rt
```

といった名前が登場します。

Intel の新しいソフトウェアスタックは、従来の Intel Media SDK から oneVPL へ移行しています。

ただし Rocky Linux 9 の FFmpeg で何を入れるべきかは、**使用している FFmpeg がどちらに対してビルドされているか確認することが重要**です。

今回使用した RPM Fusion の FFmpeg 5.1.10 は、

```bash
ffmpeg -version
```

を見ると、

```text
--enable-libmfx
```

でビルドされていました。

そして実際に、

```bash
rpm -qa | grep -Ei 'libva|intel-media|libmfx|vpl'
```

を確認すると、

```text
libva-2.22.0-1.el9.x86_64
intel-mediasdk-21.3.5-1.el9.x86_64
libva-utils-2.11.1-1.el9.x86_64
```

となっていました。

この環境では FFmpeg が `libmfx`、つまり Intel Media SDK 経由の QSV を使用する構成です。

したがって、**「Rocky 9 なら必ず libvpl + intel-vpl-gpu-rt をインストールする」と考えないほうが安全**です。

まず、

```bash
ffmpeg -version
```

の configure オプションを確認してください。

---

# 10. FFmpeg をインストールする

RPM Fusion から FFmpeg を入れます。

```bash
sudo dnf install -y ffmpeg
```

確認します。

```bash
ffmpeg -version
```

---

# 11. FFmpeg が QSV 対応か確認する

まずハードウェアアクセラレーション一覧を確認します。

```bash
ffmpeg -hwaccels
```

今回の FFmpeg では、

```text
Hardware acceleration methods:
vdpau
cuda
vaapi
qsv
drm
opencl
vulkan
```

となっていました。

ここで、

```text
vaapi
qsv
```

が確認できます。

次に QSV エンコーダを確認します。

```bash
ffmpeg -hide_banner -encoders | grep -E 'qsv|vaapi'
```

今回の環境では、

```text
V..... h264_qsv     H.264 / AVC ... (Intel Quick Sync Video acceleration)
V....D h264_vaapi   H.264/AVC (VAAPI)
V..... hevc_qsv     HEVC (Intel Quick Sync Video acceleration)
V....D hevc_vaapi   H.265/HEVC (VAAPI)
V..... mjpeg_qsv    MJPEG (Intel Quick Sync Video acceleration)
V....D mjpeg_vaapi  MJPEG (VAAPI)
V..... mpeg2_qsv    MPEG-2 video (Intel Quick Sync Video acceleration)
V....D mpeg2_vaapi  MPEG-2 (VAAPI)
V....D vp8_vaapi    VP8 (VAAPI)
V....D vp9_vaapi    VP9 (VAAPI)
V..... vp9_qsv      VP9 video (Intel Quick Sync Video acceleration)
```

となりました。

**`h264_qsv` が一覧にあることと、実際に GPU が使用できることは別問題です。**

---

# 12. まず `vainfo` を正常にする

QSV のトラブルシューティングでは、いきなり FFmpeg のオプションを変更し続けるより、

```bash
vainfo --display drm --device /dev/dri/renderD128
```

を最初に正常化したほうが早いです。

今回の環境では、

```text
libva info: VA-API version 1.22.0
libva info: Trying to open /usr/lib64/dri/iHD_drv_video.so
libva info: va_openDriver() returns -1
libva info: Trying to open /usr/lib64/dri/i965_drv_video.so
libva info: va_openDriver() returns -1
vaInitialize failed with error code -1 (unknown libva error),exit
```

となっていました。

GPU は認識され、`i915` も使用され、`/dev/dri/renderD128` も存在しているため、問題は `libva` / Intel Media Driver 付近にあると切り分けられます。

---

# 13. `iHD_drv_video.so` がどの RPM に含まれるか確認する

```bash
ls -l /usr/lib64/dri/iHD_drv_video.so
```

さらに、

```bash
rpm -qf /usr/lib64/dri/iHD_drv_video.so
```

で、どの RPM が提供しているファイルか確認できます。

RPM が分からなければ、

```bash
dnf provides '*/iHD_drv_video.so'
```

でも検索できます。

期待するのは `intel-media-driver` 系の RPM です。

```bash
sudo dnf install -y intel-media-driver
```

ここで注意したいのが `intel-mediasdk` と `intel-media-driver` は別物だということです。

---

# 14. ドライバの依存ライブラリを確認する

```bash
ldd /usr/lib64/dri/iHD_drv_video.so
```

ここに、

```text
not found
```

があれば、その依存ライブラリが不足しています。

---

# 15. パッケージの出所とバージョンも確認する

```bash
rpm -qi libva libva-utils intel-mediasdk intel-media-driver
```

または、

```bash
dnf repoquery --installed \
  --qf '%{name} %{version}-%{release} %{repoid}' \
  libva libva-utils intel-mediasdk intel-media-driver
```

を使用します。

---

# 16. iHD を明示してテストする

```bash
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

正常なら、

```text
Trying to open /usr/lib64/dri/iHD_drv_video.so
Found init function ...
va_openDriver() returns 0
```

となります。

---

# 17. QSV 単体テストをする

`vainfo` が正常になったら、入力動画とは無関係なテスト映像で QSV エンコードを確認します。

```bash
ffmpeg \
  -qsv_device /dev/dri/renderD128 \
  -f lavfi \
  -i testsrc2=size=1280x720:rate=30 \
  -t 5 \
  -c:v h264_qsv \
  -global_quality 23 \
  -f null -
```

これが成功すれば H.264 QSV エンコード経路が動作していると判断できます。

---

# 18. VAAPI エンコードでも切り分ける

```bash
ffmpeg \
  -vaapi_device /dev/dri/renderD128 \
  -f lavfi \
  -i testsrc2=size=1280x720:rate=30 \
  -vf 'format=nv12,hwupload' \
  -t 5 \
  -c:v h264_vaapi \
  -f null -
```

| VAAPI | QSV | 考えられる原因 |
|---|---|---|
| NG | NG | VA-API / Intel Media Driver / GPU デバイス側 |
| OK | NG | Media SDK / oneVPL / QSV runtime 側 |
| OK | OK | GPU スタックは正常。元の FFmpeg コマンドを調査 |
| NG | OK | 通常はあまりない特殊な構成 |

---

# 19. QSV で H.264 をエンコードする

```bash
ffmpeg \
  -qsv_device /dev/dri/renderD128 \
  -i input.mp4 \
  -c:v h264_qsv \
  -global_quality 23 \
  -c:a copy \
  output.mp4
```

---

# 20. HEVC / H.265 を QSV でエンコードする

```bash
ffmpeg \
  -qsv_device /dev/dri/renderD128 \
  -i input.mp4 \
  -c:v hevc_qsv \
  -global_quality 25 \
  -c:a copy \
  output.mp4
```

---

# 21. QSV decode + QSV encode

```bash
ffmpeg \
  -qsv_device /dev/dri/renderD128 \
  -hwaccel qsv \
  -hwaccel_output_format qsv \
  -i input.mp4 \
  -c:v h264_qsv \
  -global_quality 23 \
  -c:a copy \
  output.mp4
```

トラブルシューティングでは、まず CPU decode + QSV encode から始めるほうが切り分けやすくなります。

---

# 22. `yuv420p` → `nv12` の警告は致命的エラーではない

```text
Incompatible pixel format 'yuv420p' for codec 'h264_qsv',
auto-selecting format 'nv12'
```

これは今回の VA-API エラーとは別問題です。

必要なら、

```bash
-pix_fmt nv12
```

を指定できます。

---

# 23. ALSA の `Thread message queue blocking` も別問題

```text
[alsa] Thread message queue blocking;
consider raising the thread_queue_size option
```

が出る場合は ALSA 入力の前に、

```bash
-thread_queue_size 1024
```

などを指定します。

---

# 24. Rocky Linux 9 のセットアップ例

```bash
sudo dnf install -y epel-release

sudo dnf install -y \
  https://download1.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm \
  https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm

sudo dnf install -y \
  ffmpeg \
  libva \
  libva-utils \
  intel-media-driver
```

今回のように `ffmpeg -version` に、

```text
--enable-libmfx
```

がある環境では、

```bash
sudo dnf install -y intel-mediasdk
```

も候補になります。

Gemini Lake + RPM Fusion FFmpeg 5.1 系であれば、概ね、

```bash
sudo dnf install -y \
  ffmpeg \
  libva \
  libva-utils \
  intel-media-driver \
  intel-mediasdk
```

という構成から始めるのが分かりやすいでしょう。

---

# 25. 最終確認チェックリスト

```bash
# 0. nomodeset
cat /proc/cmdline

# 1. GPU / kernel driver
lspci -nnk | grep -A4 -Ei 'VGA|Display'

# 2. Kernel module
lsmod | grep -E 'i915|xe'

# 3. DRM
ls -l /dev/dri/

# 4. Intel Media Driver
rpm -q intel-media-driver

# 5. iHD driver
ls -l /usr/lib64/dri/iHD_drv_video.so
rpm -qf /usr/lib64/dri/iHD_drv_video.so

# 6. VA-API
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128

# 7. FFmpeg HW acceleration
ffmpeg -hwaccels

# 8. QSV encoders
ffmpeg -hide_banner -encoders | grep _qsv

# 9. QSV decoders
ffmpeg -hide_banner -decoders | grep _qsv

# 10. QSV encode test
ffmpeg \
  -qsv_device /dev/dri/renderD128 \
  -f lavfi \
  -i testsrc2=size=1280x720:rate=30 \
  -t 5 \
  -c:v h264_qsv \
  -global_quality 23 \
  -f null -
```

確認順序は次のように考えると分かりやすくなります。

```text
nomodeset の有無
↓
Intel GPU
↓
i915 / xe
↓
/dev/dri/renderD128
↓
VA-API / libva
↓
Intel Media Driver
↓
Media SDK / oneVPL
↓
FFmpeg
```

---

# 今回の Gemini Lake / UHD Graphics 605 で分かったこと

今回の実機では、

```text
Intel Corporation GeminiLake [UHD Graphics 605]
Kernel driver in use: i915
```

であり、`/dev/dri/renderD128` も存在していました。

さらに FFmpeg は `--enable-libmfx` 付きで、`qsv` / `vaapi` を認識し、`h264_qsv`、`hevc_qsv`、`vp9_qsv` なども列挙できました。

しかし、

```bash
vainfo --display drm --device /dev/dri/renderD128
```

では、

```text
Trying to open /usr/lib64/dri/iHD_drv_video.so
va_openDriver() returns -1

Trying to open /usr/lib64/dri/i965_drv_video.so
va_openDriver() returns -1
```

となっていました。

QSV テストでも、

```text
Failed to initialise VAAPI connection
Device creation failed
```

となりました。

このことから、問題は FFmpeg のエンコードオプションではなく、VA-API / Intel Media Driver 層にあると切り分けられます。

このような場合は、

```bash
rpm -qf /usr/lib64/dri/iHD_drv_video.so
ldd /usr/lib64/dri/iHD_drv_video.so

LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

の順で Intel Media Driver の導入状態・依存関係を調べます。

---

# まとめ

Rocky Linux 9 で FFmpeg + Intel QSV を使う場合、単に `ffmpeg -encoders` に `h264_qsv` が表示されれば完了、というわけではありません。

必要になるレイヤは概ね次の通りです。

| コンポーネント | 必要性 | 役割 |
|---|---|---|
| `nomodeset` 無効化 | 重要 | DRM/KMS を正常に初期化 |
| Intel GPU | 必須 | ハードウェア |
| `i915` / `xe` | 必須 | Kernel GPU driver |
| `/dev/dri/renderD128` | 必須 | DRM render node |
| `libva` | Linux QSV 環境で重要 | VA-API |
| `intel-media-driver` | 対応 Intel GPU で重要 | `iHD_drv_video.so` |
| `intel-mediasdk` | libmfx 構成 | Legacy QSV runtime |
| `libvpl` | oneVPL 構成 | oneVPL dispatcher |
| `intel-vpl-gpu-rt` | 対応する新世代 GPU | oneVPL GPU implementation |
| QSV 対応 FFmpeg | 必須 | `h264_qsv` / `hevc_qsv` 等 |

特に、

```text
Failed to initialise VAAPI connection
Failed to create a VAAPI device
```

が出た場合には、エンコードオプションを変更する前に、

```bash
cat /proc/cmdline
vainfo --display drm --device /dev/dri/renderD128
```

を確認するのが近道です。

`nomodeset` が残っていれば外し、`vainfo` が正常に動かなければ VA-API / Intel Media Driver の問題を先に解決します。

Rocky Linux 9 で QSV を構築するときは、

```text
nomodeset
↓
GPU
↓
Kernel driver
↓
DRM
↓
VA-API
↓
Intel Media Driver
↓
Media SDK / oneVPL
↓
FFmpeg
```

というレイヤを意識して、**下から一段ずつ正常動作を確認する**のが最も確実なセットアップ方法です。
