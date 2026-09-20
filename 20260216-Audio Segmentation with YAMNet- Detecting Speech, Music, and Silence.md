---
pubDatetime: 2026-02-16T16:41:21+09:00
title: "YAMNetを用いた音声セグメンテーション：Speech・Music・Silenceの検出"
description: "全体構成、必要ライブラリ、設定パラメータを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

本記事では、音声ファイルを解析し、**Speech（音声）**、**Music（音楽）**、**Silence（無音）** に自動分類して時間区間ごとに分割するPythonプログラムについて解説します。

本システムは、**音量ベースの判定**と**深層学習モデル**を組み合わせ、さらに境界補正を行うことで精度の高いセグメンテーションを実現します。

---

## 全体構成

処理は以下の5段階で構成されています。

1. 音声の読み込みと前処理
2. 音量による無音判定
3. YAMNetによる分類
4. 粗い（Coarse）セグメンテーション
5. 境界の微調整（Fine refinement）

最終的に、音声ファイルを時間付きラベルの区間として出力します。

---

## 必要ライブラリ

```bash
pip install tensorflow tensorflow-hub librosa numpy
```

### 使用ライブラリ

* **TensorFlow**：深層学習フレームワーク
* **TensorFlow Hub**：学習済みYAMNetモデルの読み込み
* **Librosa**：音声読み込み・処理
* **NumPy**：数値計算

---

## 設定パラメータ

### 音声設定

* サンプリングレート：16,000 Hz
* モノラル変換

### 粗いセグメント分割

* 1チャンク：8秒

### 境界微調整設定

* 探索範囲：±0.8秒
* 微調整ウィンドウ長：0.96秒
* ホップサイズ：0.48秒

### 音量閾値

* Silence：−55 dBFS未満
* 低ノイズ：−45 dBFS未満

---

## YAMNetモデルの読み込み

TensorFlow Hubから学習済みのYAMNetモデルを読み込みます。

```python
yamnet = hub.load("https://tfhub.dev/google/yamnet/1")
```

YAMNetは多数の音イベント分類を出力します。その中から以下のカテゴリを抽出します。

### Speech系クラス

* Speech
* Conversation
* Narration
* Singing
* Whispering
* Shout
* など

### Music系クラス

* Music
* Musical instrument

出力スコアは時間方向に平均化され、以下を算出します。

* speech_score
* music_score

---

## 音量による無音判定

まず音量を計算します。

* RMS（実効値）
* Peak（最大振幅）

これらをdBFS（フルスケール基準デシベル）に変換します。

### 判定ロジック

* RMS < −55 dBFS → `"Silence"`
* RMS < −45 dBFS → `"Silence2"`
* それ以外 → YAMNetで分類

非常に静かな区間では深層学習推論を省略し、計算効率を向上させます。

---

## チャンク分類処理

各チャンクについて：

1. 音量チェック
2. 無音でなければYAMNet推論
3. speech_score と music_score を比較

```python
label = "Speech" if speech_score >= music_score else "Music"
```

出力内容：

* ラベル
* RMSレベル
* Peakレベル
* Speechスコア
* Musicスコア

---

## 粗いセグメンテーション

音声全体を**8秒単位**に分割し、それぞれ独立に分類します。

出力例：

```
00:00:00 - 00:00:08  Speech
00:00:08 - 00:00:16  Music
```

---

## 連続区間の統合

同じラベルが連続するチャンクは統合します。

例：

| 区間     | ラベル    |
| ------ | ------ |
| 0–8秒   | Speech |
| 8–16秒  | Speech |
| 16–24秒 | Music  |

統合後：

* 0–16秒 → Speech
* 16–24秒 → Music

より自然な区間分割になります。

---

## 境界の微調整

粗い分割では境界が正確でない可能性があります。

そこで：

1. ラベルが変わる境界付近（±0.8秒）を再探索
2. 小さなウィンドウ（0.96秒）で再分類
3. 左側が前ラベル、右側が次ラベルになる最適位置を探索

この最適化により、SpeechとMusicの切り替わり位置がより正確になります。

---

## 時刻フォーマット

秒数は以下形式に変換されます。

```
HH:MM:SS
```

---

## 出力例

### 粗いセグメント

```
=== Segments (Coarse) ===
00:00:00 - 00:00:16  Speech
00:00:16 - 00:00:32  Music
```

### 微調整後セグメント

```
=== Segments (Refined) ===
00:00:00 - 00:00:14  Speech
00:00:14 - 00:00:32  Music
```

境界が補正されていることが確認できます。

---

## 実行方法

```bash
python script.py input_audio.wav
```

処理内容：

1. 音声読み込み
2. 粗いセグメント分類
3. 境界微調整
4. 結果出力

---

## まとめ

本システムは以下を組み合わせています。

* 信号処理による無音判定
* YAMNetによる深層学習分類
* 2段階セグメンテーション（粗い＋微調整）

この構成により、計算効率と分類精度の両立を実現しています。

応用例：

* ポッドキャスト構造解析
* 音声アーカイブ自動分類
* メディア編集前処理
* コンテンツ自動インデックス化

信号処理とニューラルネットワーク推論を統合した、実用的な音声セグメンテーション手法です。

`pip install tensorflow tensorflow-hub librosa numpy`

```python
import numpy as np 
import librosa 
import tensorflow as tf 
import tensorflow_hub as hub 
from sys import argv 
 
# ---------------------------- 
# Configuration 
# ---------------------------- 
TARGET_SR = 16000 
 
# Coarse window 
CHUNK_SEC = 8.0 
 
# Fine boundary refinement 
FINE_SEARCH_RADIUS_SEC = 0.8 
FINE_WIN_SEC = 0.96 
FINE_HOP_SEC = 0.48 
 
# Volume thresholds 
SILENCE_DBFS = -55.0 
LOW_NOISE_DBFS = -45.0 
 
# Speech / Music thresholds 
SPEECH_TH = 0.20 
MUSIC_TH = 0.20 
 
# ---------------------------- 
# Load YAMNet 
# ---------------------------- 
yamnet = hub.load("https://tfhub.dev/google/yamnet/1") 
 
class_map_path = yamnet.class_map_path().numpy().decode("utf-8") 
class_names = [] 
with open(class_map_path, "r", encoding="utf-8") as f: 
    next(f) 
    for line in f: 
        parts = line.strip().split(",") 
        class_names.append(parts[2]) 
 
SPEECH_CLASSES = { 
    "Speech", 
    "Child speech, kid speaking", 
    "Conversation", 
    "Narration, monologue", 
    "Babbling", 
    "Speech synthesizer", 
    "Shout", 
    "Yell", 
    "Whispering", 
    "Singing", 
} 
 
MUSIC_CLASSES = { 
    "Music", 
    "Musical instrument", 
} 
 
name_to_idx = {n: i for i, n in enumerate(class_names)} 
speech_idxs = [name_to_idx[n] for n in SPEECH_CLASSES if n in name_to_idx] 
music_idxs = [name_to_idx[n] for n in MUSIC_CLASSES if n in name_to_idx] 
 
# ---------------------------- 
# Volume utilities 
# ---------------------------- 
def rms_dbfs(x): 
    rms = np.sqrt(np.mean(np.square(x)) + 1e-12) 
    return 20.0 * np.log10(rms + 1e-12) 
 
def peak_dbfs(x): 
    peak = np.max(np.abs(x)) + 1e-12 
    return 20.0 * np.log10(peak + 1e-12) 
 
def classify_volume_level(x): 
    r = rms_dbfs(x) 
    p = peak_dbfs(x) 
 
    if r < SILENCE_DBFS: 
        return "Silence", r, p 
    if r < LOW_NOISE_DBFS: 
        return "Silence2", r, p 
    return None, r, p 
 
# ---------------------------- 
# YAMNet classification 
# ---------------------------- 
def classify_chunk_with_yamnet(chunk_16k): 
    waveform = tf.convert_to_tensor(chunk_16k, dtype=tf.float32) 
    scores, embeddings, spectrogram = yamnet(waveform) 
    mean_scores = scores.numpy().mean(axis=0) 
 
    speech_score = float(mean_scores[speech_idxs].sum()) if speech_idxs else 0.0 
    music_score = float(mean_scores[music_idxs].sum()) if music_idxs else 0.0 
 
    return speech_score, music_score 
 
def decide_label(chunk_16k): 
    volume_label, r_db, p_db = classify_volume_level(chunk_16k) 
 
    if volume_label is not None: 
        return { 
            "label": volume_label, 
            "rms_dbfs": r_db, 
            "peak_dbfs": p_db, 
            "speech_score": 0.0, 
            "music_score": 0.0, 
        } 
 
    speech_score, music_score = classify_chunk_with_yamnet(chunk_16k) 
 
    label = "Speech" if speech_score >= music_score else "Music" 
 
    return { 
        "label": label, 
        "rms_dbfs": r_db, 
        "peak_dbfs": p_db, 
        "speech_score": speech_score, 
        "music_score": music_score, 
    } 
 
# ---------------------------- 
# Window slicing 
# ---------------------------- 
def slice_window(y, start_s, win_s): 
    start = int(round(start_s * TARGET_SR)) 
    win = int(round(win_s * TARGET_SR)) 
 
    if start < 0: 
        pad_left = -start 
        start = 0 
    else: 
        pad_left = 0 
 
    end = start + win 
    chunk = y[start:end] 
 
    if pad_left > 0: 
        chunk = np.pad(chunk, (pad_left, 0)) 
    if len(chunk) < win: 
        chunk = np.pad(chunk, (0, win - len(chunk))) 
 
    return chunk 
 
# ---------------------------- 
# Coarse pass 
# ---------------------------- 
def coarse_pass(y): 
    chunk_len = int(CHUNK_SEC * TARGET_SR) 
    n_chunks = int(np.ceil(len(y) / chunk_len)) 
    out = [] 
 
    for i in range(n_chunks): 
        start = i * chunk_len 
        end = min((i + 1) * chunk_len, len(y)) 
        chunk = y[start:end] 
 
        if len(chunk) < chunk_len: 
            chunk = np.pad(chunk, (0, chunk_len - len(chunk))) 
 
        result = decide_label(chunk) 
 
        out.append({ 
            "chunk_index": i, 
            "start_sec": i * CHUNK_SEC, 
            "end_sec": (i + 1) * CHUNK_SEC, 
            **result, 
        }) 
 
    return out 
 
# ---------------------------- 
# Build segments 
# ---------------------------- 
def build_segments(coarse): 
    if not coarse: 
        return [] 
 
    segs = [{ 
        "label": coarse[0]["label"], 
        "start": coarse[0]["start_sec"], 
        "end": coarse[0]["end_sec"] 
    }] 
 
    for r in coarse[1:]: 
        if r["label"] == segs[-1]["label"]: 
            segs[-1]["end"] = r["end_sec"] 
        else: 
            segs.append({ 
                "label": r["label"], 
                "start": r["start_sec"], 
                "end": r["end_sec"] 
            }) 
 
    return segs 
 
# ---------------------------- 
# Fine boundary refinement 
# ---------------------------- 
def refine_single_boundary(y, t0, left_label, right_label): 
    R = FINE_SEARCH_RADIUS_SEC 
    win = FINE_WIN_SEC 
    hop = FINE_HOP_SEC 
 
    start_min = t0 - R 
    start_max = t0 + R - win 
    if start_max < start_min: 
        return t0 
 
    starts = [] 
    labels = [] 
 
    s = start_min 
    while s <= start_max + 1e-9: 
        chunk = slice_window(y, s, win) 
        r = decide_label(chunk) 
        starts.append(s) 
        labels.append(r["label"]) 
        s += hop 
 
    if len(labels) < 2: 
        return t0 
 
    best_k = None 
    best_score = -1 
 
    left_prefix = [0] * (len(labels) + 1) 
    for i, lab in enumerate(labels): 
        left_prefix[i + 1] = left_prefix[i] + (1 if lab == left_label else 0) 
 
    right_suffix = [0] * (len(labels) + 1) 
    for i in range(len(labels) - 1, -1, -1): 
        right_suffix[i] = right_suffix[i + 1] + (1 if labels[i] == right_label else 0) 
 
    for k in range(len(labels) - 1): 
        score = left_prefix[k + 1] + right_suffix[k + 1] 
        if score > best_score: 
            best_score = score 
            best_k = k 
 
    if best_k is None: 
        return t0 
 
    return starts[best_k] + win 
 
def refine_segments_with_finepass(y, segments): 
    if len(segments) < 2: 
        return segments 
 
    refined = [dict(segments[0])] 
 
    for i in range(1, len(segments)): 
        prev_seg = refined[-1] 
        cur_seg = dict(segments[i]) 
 
        t0 = prev_seg["end"] 
        refined_t = refine_single_boundary( 
            y, t0, 
            prev_seg["label"], 
            cur_seg["label"] 
        ) 
 
        refined_t = max(prev_seg["start"], min(refined_t, cur_seg["end"])) 
 
        prev_seg["end"] = refined_t 
        cur_seg["start"] = refined_t 
 
        refined.append(cur_seg) 
 
    return refined 
 
# ---------------------------- 
# Time formatting 
# ---------------------------- 
def sec_to_hms(sec): 
    total = int(round(sec)) 
    h = total // 3600 
    m = (total % 3600) // 60 
    s = total % 60 
    return f"{h:02d}:{m:02d}:{s:02d}" 
 
# ---------------------------- 
# Main 
# ---------------------------- 
if __name__ == "__main__": 
    audio_path = argv[1] 
 
    y, sr = librosa.load(audio_path, sr=TARGET_SR, mono=True) 
 
    coarse = coarse_pass(y) 
    segments = build_segments(coarse) 
    refined_segments = refine_segments_with_finepass(y, segments) 
 
    print("=== Segments (Coarse) ===") 
    for s in segments: 
        print(f"{sec_to_hms(s['start'])} - {sec_to_hms(s['end'])}  {s['label']}") 
 
    print("\n=== Segments (Refined) ===") 
    for s in refined_segments: 
        print(f"{sec_to_hms(s['start'])} - {sec_to_hms(s['end'])}  {s['label']}") 
```
