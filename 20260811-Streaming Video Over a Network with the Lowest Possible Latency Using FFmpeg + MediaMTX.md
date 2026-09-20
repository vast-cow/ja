---
title: "FFmpeg + MediaMTXで映像をできるだけ低遅延にネットワーク配信する"
description: ""
pubDatetime: 2026-08-11T06:54:54.962Z
updatedDate: 2026-08-12T04:18:56.930Z
---

HDMIキャプチャなどから取り込んだ映像をLAN内の別PCへ送りたいとき、意外と問題になるのが**映像の遅延**です。

普通にH.264へエンコードしてネットワーク配信すると、

* キャプチャ
* FFmpeg内部のキュー
* エンコード
* マルチプレクサ
* ネットワーク
* プレイヤー側のバッファ
* デコード・表示

といった複数の段階で少しずつバッファリングされ、最終的には数百ms〜数秒の遅延になることがあります。

そこで今回は、

**V4L2 + ALSA → FFmpeg → MediaMTX → RTSP/UDP → ffplay**

という構成で、可能な限りバッファを減らした低遅延ストリーミングを構築します。

FFmpegの現行ドキュメントでも、`nobuffer` は入力解析時のバッファリングによる遅延を減らすためのオプション、`low_delay` は低遅延動作を強制するフラグとして定義されています。

---

## 構成

今回の構成は次のようになります。

```text
HDMI入力
   ↓
キャプチャデバイス
 /dev/video0
   ↓
FFmpeg
  ├─ V4L2で映像入力
  ├─ ALSAで音声入力
  ├─ Intel QSVでH.264エンコード
  ↓
RTSP / UDP
   ↓
MediaMTX
   ↓
LAN
   ↓
ffplay
   ↓
画面表示
```

MediaMTXには映像そのものをエンコードさせるのではなく、**RTSPサーバーとして中継を担当させます**。

MediaMTXはRTSPを含むリアルタイム映像・音声ストリームのpublish/readに対応するメディアサーバーで、外部コマンドをhookとして起動することもできます。

今回はその`runOnDemand`機能を使います。

---

# MediaMTXの設定

`mediamtx.yml`を次のようにしています。

```yaml
paths:
  hdmi:
    runOnDemand: >-
      ffmpeg -hide_banner -y
      -loglevel warning
      -fflags nobuffer
      -flags low_delay
      -init_hw_device qsv=qsv
      -filter_hw_device qsv
      -thread_queue_size 4
      -f alsa
      -ac 2
      -ar 48000
      -i hw:1,0
      -thread_queue_size 1
      -f v4l2
      -input_format yuyv422
      -video_size 1920x1080
      -framerate 30
      -i /dev/video0
      -map 1:v:0
      -map 0:a:0
      -vf 'hwupload=extra_hw_frames=0,vpp_qsv=format=nv12'
      -c:v h264_qsv
      -preset veryfast
      -global_quality:v 35
      -async_depth 1
      -bf 0
      -g 30
      -c:a libopus -b:a 192k
      -f rtsp
      -rtsp_transport udp
      -muxdelay 0
      rtsp://127.0.0.1:8554/hdmi
    runOnDemandRestart: yes
    runOnDemandStartTimeout: 10s
    runOnDemandCloseAfter: 5s
```

一見するとオプションがかなり多いですが、低遅延化という観点ではいくつかのポイントに分けて考えると分かりやすくなります。

---

# `runOnDemand`で必要なときだけFFmpegを起動する

まずMediaMTX側です。

```yaml
runOnDemand: >-
  ffmpeg ...
```

`runOnDemand`は、クライアントからそのpathへのアクセスが発生したときに外部コマンドを起動する機能です。

つまり、

```text
誰も見ていない
↓
FFmpeg停止

ffplayから /hdmi に接続
↓
MediaMTXがFFmpegを起動
↓
FFmpegが /hdmi にpublish
↓
再生開始
```

という動作になります。

MediaMTX公式ドキュメントでも、`runOnDemand`に指定したコマンドはreaderからpathが要求されたタイミングで開始される仕組みになっています。

常時FFmpegを動かしておく必要がないため、HDMI配信を必要なときだけ使いたい場合に便利です。

さらに、

```yaml
runOnDemandRestart: yes
```

としているので、FFmpegが何らかの理由で終了した場合にも再起動されます。現行のMediaMTX設定リファレンスにも、このオプションはコマンド終了時に再起動する設定として定義されています。

---

# 入力側のバッファを極力小さくする

低遅延ストリーミングで重要なのが、

**「溜めてから処理する」のを避けること**

です。

そのためFFmpegの冒頭で、

```text
-fflags nobuffer
-flags low_delay
```

を指定しています。

## `-fflags nobuffer`

```text
-fflags nobuffer
```

は入力ストリーム解析時に発生するバッファリングによる遅延を減らすための設定です。FFmpeg公式ドキュメントでも、初期入力解析時のバッファリングによるレイテンシーを削減するオプションとされています。

リアルタイム入力では、

```text
できるだけパケットを貯めない
↓
届いたデータをすぐ次の処理へ渡す
```

という方向に設定しておくことが重要です。

## `-flags low_delay`

```text
-flags low_delay
```

も名前の通り、コーデック処理を低遅延方向へ設定するためのフラグです。FFmpegでは`low_delay`が「Force low delay」と定義されています。

---

# 映像と音声を別デバイスから入力する

今回の構成では音声をALSA、

```text
-f alsa
-ac 2
-ar 48000
-i hw:1,0
```

映像をV4L2、

```text
-f v4l2
-input_format yuyv422
-video_size 1920x1080
-framerate 30
-i /dev/video0
```

から取り込んでいます。

したがってFFmpegから見ると入力は、

```text
input 0 = ALSA
input 1 = V4L2
```

になります。

出力するストリームを、

```text
-map 1:v:0
-map 0:a:0
```

と明示しています。

つまり、

```text
映像 → input 1のvideo 0
音声 → input 0のaudio 0
```

です。

---

# `thread_queue_size`を小さくする

入力にはそれぞれ、

```text
-thread_queue_size 4
```

```text
-thread_queue_size 1
```

を指定しています。

`thread_queue_size`は、入力デバイスなどから読み込んだパケットをFFmpeg内部で何個までキューに保持するかを決める設定です。FFmpeg公式ドキュメントでも、入力の場合はデバイスやファイルから読み込む際のqueued packetsの最大数と説明されています。

キューを大きくすると、

```text
多少処理が詰まる
↓
キューへ蓄積
↓
フレームを捨てず処理
```

しやすくなります。

一方で低遅延用途では、

```text
処理が追いつかない
↓
過去の映像がキューへ溜まる
↓
表示がどんどん現実時間から遅れる
```

という問題になります。

そのため今回はかなり小さい値にしています。

これは、

**フレームを絶対に落とさないことより、現在時刻に近い映像を表示することを優先する**

という設計です。

低遅延ストリーミングでは非常に重要な考え方です。

---

# Intel Quick Sync Videoでエンコードする

1920×1080 30fpsの映像をH.264へソフトウェアエンコードすると、CPU負荷によっては処理そのものが遅延原因になります。

そこでIntel Quick Sync Video、いわゆるQSVを利用しています。

```text
-init_hw_device qsv=qsv
-filter_hw_device qsv
```

そして映像を、

```text
-vf 'hwupload=extra_hw_frames=0,vpp_qsv=format=nv12'
```

でQSV側へアップロードし、NV12へ変換します。

エンコーダーは、

```text
-c:v h264_qsv
```

です。

これによってH.264エンコードをIntel GPUのハードウェアエンコーダーへ担当させます。

---

# エンコーダーでも「先読み」を減らす

ここからが低遅延化の重要部分です。

```text
-preset veryfast
-global_quality:v 35
-async_depth 1
-bf 0
-g 30
```

としています。

## `-preset veryfast`

```text
-preset veryfast
```

QSVのpresetは、

```text
veryfast
faster
fast
medium
slow
slower
veryslow
```

という選択肢があり、FFmpegでは`veryfast`側が速度優先、`veryslow`側が画質優先として定義されています。

ライブ配信ではエンコード品質を限界まで高めるより、

**1フレームを素早くエンコードしてネットワークへ送る**

ことを優先します。

そのため`veryfast`を選択しています。

---

# `async_depth 1`が重要

```text
-async_depth 1
```

も低遅延化に効く設定です。

QSVは複数フレームを非同期に処理することでスループットを高められます。

ただし、

```text
複数フレームを並列処理
```

するということは、逆に言えばエンコーダー内部に複数フレームが存在することにもなります。

FFmpegのQSVドキュメントでも`async_depth`は非同期処理数に関係するパラメータとして定義されており、QSV decoderについては値を増やすほどレイテンシーも増えると明記されています。

そこで、

```text
-async_depth 1
```

まで下げています。

狙っているのは、

```text
フレーム入力
↓
エンコード
↓
すぐ出力
```

という単純なパイプラインです。

スループットよりlatencyを優先しています。

---

# Bフレームを使わない

低遅延H.264では、

```text
-bf 0
```

が特に重要です。

Bフレームを使用すると、あるフレームをエンコード・デコードする際に未来側のフレームを参照することがあります。

概念的には、

```text
I P B B P
```

のような構造です。

そのためエンコーダーやデコーダーでフレーム並び替えが必要になり、低遅延用途では不利になります。

そこで、

```text
-bf 0
```

としてBフレームを完全に無効化しています。

圧縮効率は多少犠牲になりますが、リアルタイム用途ではこちらの方が扱いやすくなります。

---

# GOPを1秒にする

```text
-g 30
```

もポイントです。

今回は、

```text
-framerate 30
```

なので30fpsです。

したがって、

```text
30フレーム ÷ 30fps = 1秒
```

となり、おおよそ1秒ごとにGOPが区切られます。

FFmpegでは`-g`はGOP sizeを指定するパラメータです。

GOPを短くすると圧縮効率は多少悪くなりますが、

* 再生開始
* パケットロス後の復帰
* ストリームへの途中参加

といった場面で扱いやすくなります。

低遅延用途では圧縮率だけでなく、**素早く現在の映像へ追いつけること**も重要です。

---

# `global_quality:v 35`

```text
-global_quality:v 35
```

ではQSVの品質を指定しています。

`h264_qsv`で`global_quality:v`を指定すると、条件に応じてICQなどの品質ベースのレート制御が利用されます。FFmpegのQSVドキュメントではICQ時の範囲は1〜51で、**1が最高品質**とされています。

つまり、

```text
小さい値
↓
高画質・大きなデータ量

大きい値
↓
低画質・小さなデータ量
```

という方向です。

`35`はかなり圧縮寄りの設定なので、ネットワーク帯域や画質を見ながら調整できます。

たとえば画質を上げるなら、

```text
-global_quality:v 28
```

などへ下げて比較すると分かりやすいでしょう。

---

# 音声はOpus 192kbps

音声は、

```text
-c:a libopus -b:a 192k
```

としています。

映像に比べれば音声エンコードの負荷や帯域は小さいため、Opus 192kbpsとしています。

ただし「映像だけ確認できればよい」という監視用途などでは音声を削除して、

```text
-an
```

とすることで構成をさらに単純化できます。

---

# RTSPはUDPを使う

FFmpegからMediaMTXへの出力は、

```text
-f rtsp
-rtsp_transport udp
-muxdelay 0
rtsp://127.0.0.1:8554/hdmi
```

です。

低遅延化では、

```text
-rtsp_transport udp
```

がポイントです。

FFmpegではRTSPのlower transportとしてUDPまたはTCPなどを選択できます。UDPの場合はUDP、TCPの場合はRTSP control channel内へインターリーブしてデータが流れます。

UDPにはTCPのような、

```text
パケット消失
↓
再送待ち
↓
到着するまで後続処理も待つ
```

という仕組みがありません。

そのため、

**多少映像が乱れてもよいので、古い映像を待たず現在の映像を出す**

というリアルタイム用途と相性がよい方式です。

ただし当然ながら、

* Wi-Fi
* 混雑したLAN
* インターネット越し

などパケットロスが発生しやすい環境では映像が乱れる可能性があります。

逆に安定性を優先するなら、

```text
-rtsp_transport tcp
```

という選択肢もあります。

低遅延と伝送の確実性はトレードオフです。

---

# `muxdelay 0`

さらに、

```text
-muxdelay 0
```

としています。

`muxdelay`はFFmpegの出力側で使用される遅延関連パラメータです。FFmpeg公式のRTSP送信例でも`-muxdelay 0.1`を指定した例があります。

今回はさらに攻めて、

```text
-muxdelay 0
```

とし、

**パケットをまとめるために待つ時間を極力発生させない**

方向へ寄せています。

---

# MediaMTX自身にはエンコードさせない

今回の構成で重要なのは、

```text
FFmpeg
↓
rtsp://127.0.0.1:8554/hdmi
```

として、一度localhostのMediaMTXへpublishしていることです。

クライアントは、

```text
rtsp://x.x.x.x:8554/hdmi
```

へアクセスします。

つまりMediaMTXは、

```text
FFmpeg
   ↓
MediaMTX
   ↓
複数クライアント
```

という中継点になります。

この構成にすると、キャプチャデバイスやエンコーダーを各クライアントが直接触る必要がありません。

複数端末へ配信したい場合にも扱いやすくなります。

---

# ffplay側でもバッファを削る

送信側を低遅延化しても、受信側のプレイヤーが映像を1秒溜めてから再生していたら意味がありません。

そこでffplay側も、

```bash
ffplay \
  -fflags nobuffer \
  -flags low_delay \
  -framedrop \
  -rtsp_transport udp \
  rtsp://x.x.x.x:8554/hdmi
```

としています。

---

## `-fflags nobuffer`

送信側と同様に、

```text
-fflags nobuffer
```

を指定し、入力解析時のバッファリングによるレイテンシーを減らします。

---

## `-flags low_delay`

```text
-flags low_delay
```

でデコードも低遅延方向へ設定します。

---

## `-framedrop`

```text
-framedrop
```

もリアルタイム用途では重要です。

ffplayでは、映像が同期から遅れた場合にvideo frameをdropできるオプションとして`framedrop`が用意されています。

低遅延用途では、

```text
すべてのフレームを律儀に表示する
```

よりも、

```text
処理が遅れたら古いフレームを捨てる
↓
現在時刻へ追いつく
```

ほうが重要です。

これは映像品質を多少犠牲にしてでも遅延を増大させないための設定です。

---

# 低遅延配信では「捨てる」ことが重要

ここまでの設定を見ると、共通した思想があります。

それは、

**古いデータをなるべく保持しない**

ということです。

通常の動画再生では、

```text
パケットロスさせない
フレームを落とさない
映像を滑らかにする
```

ことが重要です。

しかしリアルタイム映像では事情が違います。

例えばゲーム画面やカメラ映像を表示するとき、

```text
3秒前の映像を完璧に表示
```

することより、

```text
多少フレームが飛んでも100ms前後の映像を表示
```

できるほうが有用な場合があります。

そのため今回の設定では、

```text
小さい入力キュー
↓
Bフレームなし
↓
QSVの非同期深度を削減
↓
muxerの待ち時間を削減
↓
UDP
↓
ffplayでもバッファ削減
↓
遅れたらframe drop
```

という形で、パイプラインの各段階から待ち時間を削っています。

---

# 遅延はどこで発生するのか

低遅延化するときは、ネットワークだけを見るのではなく、

```text
Capture
↓
Queue
↓
Filter
↓
Encode
↓
Mux
↓
Network
↓
Demux
↓
Decode
↓
Render
```

というパイプライン全体を見る必要があります。

例えばLANのpingが、

```text
1ms
```

だったとしても、

```text
エンコーダー 100ms
プレイヤーバッファ 500ms
```

なら、ネットワークをいくら改善しても大きな効果はありません。

むしろリアルタイム動画の場合、

**ネットワークそのものよりエンコーダー・デコーダー・プレイヤー内部のバッファが大きな遅延源になる**

ことがあります。

今回多数の低遅延オプションを指定しているのはそのためです。

---

# さらに遅延を詰めたい場合

この設定でもまだ遅延が気になる場合、いくつか試せるポイントがあります。

まずffplayのRTSP受信では、UDPパケットの並び替え用バッファが存在します。FFmpegドキュメントでは、UDP受信時のpacket reorderingを`max_delay`を0にすることで無効化できるとされています。

例えば、

```bash
ffplay \
  -fflags nobuffer \
  -flags low_delay \
  -framedrop \
  -rtsp_transport udp \
  -max_delay 0 \
  rtsp://x.x.x.x:8554/hdmi
```

という設定も試せます。

ただしこれはパケット順序の乱れに対する耐性をさらに削るため、ネットワーク品質によっては映像が不安定になります。

また、

```text
-g 30
```

を、

```text
-g 15
```

などへ縮めることもできます。

ただしGOPを短くするほど一般に圧縮効率は悪化します。

低遅延化には常に、

```text
遅延
画質
帯域
安定性
CPU/GPU負荷
```

のトレードオフがあります。

---

# UDPだから必ず低遅延、ではない

注意したいのが、

```text
UDP = 必ず低遅延
```

ではないということです。

UDPにすると再送待ちを避けやすい一方、ネットワーク品質が悪い場合は、

```text
packet loss
↓
映像破損
↓
次の復旧可能なフレームまで待つ
```

ということもあります。

したがって実際には、

**有線LAN + UDP**

のような、低ロスかつ安定したネットワークで使うのが理想的です。

Wi-Fi環境ならUDPとTCPを両方試し、

```text
実際の遅延
映像の乱れ
安定性
```

を比較したほうがよいでしょう。

---

# まとめ

今回の構成では、

```text
HDMI
↓
V4L2 / ALSA
↓
FFmpeg
↓
Intel QSV H.264
↓
RTSP / UDP
↓
MediaMTX
↓
LAN
↓
ffplay
```

というパイプラインを作りました。

低遅延化のポイントは、単に高速なエンコーダーを使用することではありません。

各段階で発生する、

**「念のため少し溜めておく」バッファを可能な限り取り除くこと**

が重要です。

今回の主要な設定をまとめると、

```text
-fflags nobuffer
    入力側のバッファリングを削減

-flags low_delay
    コーデックを低遅延方向へ

-thread_queue_size
    キャプチャ入力のキューを小さくする

-c:v h264_qsv
    Intel QSVで高速にH.264エンコード

-preset veryfast
    エンコード速度優先

-async_depth 1
    QSV内部の非同期処理深度を削減

-bf 0
    Bフレームを無効化

-g 30
    30fpsなら約1秒GOP

-rtsp_transport udp
    再送待ちを避ける方向へ

-muxdelay 0
    muxer側の待ち時間を削減

-framedrop
    再生が遅れたら古いフレームを捨てる
```

となります。

低遅延映像配信で大切なのは、

**「すべてのフレームを確実に届ける」ことと「今の映像を届ける」ことは別の目標**

だと理解することです。

録画ならフレームを失わないことが重要です。

一方、リアルタイムモニター、遠隔操作、ゲーム画面、カメラ監視などでは、古い映像が完全に届くよりも、多少フレームが欠けても最新の映像が表示されることのほうが重要です。

その方針でキャプチャ・エンコード・ネットワーク・プレイヤーのすべてを調整すると、RTSPでもかなり低遅延な映像伝送を構成できます。

<!--
-c:v h264_qsv \
-g 0 \
-bf 0 \
-adaptive_i 0 \
-async_depth 1 \
-look_ahead 0 \
-forced_idr 1
-->
