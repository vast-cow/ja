---
pubDatetime: 2026-01-29T00:04:39+09:00
title: "Google TV（Android TV OS）向け：Android Studio + Kotlin で 最小構成アプリ を起動するまで"
description: "利用環境（Android Studio）、新規プロジェクト作成、実行（エミュレータ or 実機）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

対象：**Google TV（= Android TV OS）**
目的：Android Studio + Kotlin で 最小構成アプリを起動できるところまで。

## 利用環境（Android Studio）

```text
Android Studio Otter 3 Feature Drop | 2025.2.3
Build #AI-252.28238.7.2523.14688667, built on January 9, 2026
Runtime version: 21.0.8+-14196175-b1038.72 amd64
VM: OpenJDK 64-Bit Server VM by JetBrains s.r.o.
Toolkit: sun.awt.windows.WToolkit
Windows 11.0
GC: G1 Young Generation, G1 Concurrent GC, G1 Old Generation
Memory: 2048M
Cores: 8
Registry:
  ide.experimental.ui=true
```

## 1. 新規プロジェクト作成

1. Android Studio → **New Project**
2. テンプレート：**Television → Empty Activity**
3. Language：**Kotlin**
4. **minSdk は 21 以上**（TV は API 21+ が前提）
5. 作成後、まず **Gradle Sync**（右下ステータスバーで進行状況を確認）

   * 初回はかなり時間がかかることがある


## 2. 実行（エミュレータ or 実機）

### A. TV エミュレータ（未完）

* Android Studio → Tools → **Device Manager** → **+ Create Virtual Device**
* Add Device → **TV → Television (720p)**
* Configure virtual device → API：**API 34 (Android 14 / UpsideDownCake)**
  ※ Google TV に合わせた想定
* 以降は中止したため省略



### B. Google TV 実機

#### 1) `adb` の場所を確認

1. Android Studio → Tools → **SDK Manager**
2. **Android SDK Locations** を確認
3. 例：`{Android SDK Location}/platform-tools/adb.exe`



#### 2) 実機側で Developer options を有効化

* 実機：**設定 → About → Build** を連打 → **Developer options 有効化**



#### 3) デバッグ設定（USB / Wi-Fi）

* Developer options（開発者向けオプション）

  * **USB debugging**：USB 接続の場合 ON
  * **Wireless debugging**：Wi-Fi の場合

    * 有効化（一旦無効にしてから有効にするとうまくいくようだ）
    * **「専用コードによるデバイスのペア設定」** 画面を開く
      → PINなど が表示されるので、それを使ってペアリングする


#### 4) `adb` でペアリング／接続

```bash
adb kill-server
adb start-server

# Google TV 側のPIN画面をいったん閉じて再表示してから実行
adb pair {IP}:{PORT} {PIN}

adb shell
# pair が successful でも shell に入れないことが何回もあった
```


#### 5) Android Studio から実機実行

1. Android Studio → Tools → **Device Manager**
2. 実機が追加されていることを確認
3. **Run 'app'** を実行
   → Google TV 実機でアプリが起動

## 参考

- [Create and run a TV app  |  Android TV  |  Android Developers](https://developer.android.com/training/tv/get-started/create)
- [Use Jetpack Compose on Android TV  |  Android Developers](https://developer.android.com/training/tv/playback/compose)
