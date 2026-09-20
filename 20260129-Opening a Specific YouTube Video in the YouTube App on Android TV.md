---
pubDatetime: 2026-01-29T00:18:18+09:00
title: "Android TVで指定したYouTube動画をYouTubeアプリで開く方法"
description: "この記事では、Android TV で YouTube動画をYouTubeアプリで開く方法を、Kotlin のサンプルコード付きで分かりやすく解説します。 Android TV では通常のスマホ向け YouTube アプリとはパッケージ名や挙動が異なるため、段階的なフォールバック処理が重要になります…"
---

この記事では、**Android TV で YouTube動画をYouTubeアプリで開く方法**を、Kotlin のサンプルコード付きで分かりやすく解説します。
Android TV では通常のスマホ向け YouTube アプリとはパッケージ名や挙動が異なるため、段階的なフォールバック処理が重要になります。

## Android TV 用 YouTube アプリのパッケージ名

Android TV 版 YouTube アプリのパッケージ名は次の通りです。

* `com.google.android.youtube.tv`

このパッケージを指定して `Intent` を送ることで、Android TV 上の YouTube アプリを直接起動できます。

## Kotlin で指定動画を開く実装例

以下は、動画 ID を指定して YouTube を起動する拡張関数の例です。

```kotlin
import android.content.ActivityNotFoundException
import android.content.Context
import android.content.Intent
import android.net.Uri

fun Context.openYouTubeVideoOnTv(videoId: String) {
    val pm = packageManager

    // 1) Android TV 用 YouTube アプリを直接指定（vnd.youtube: が最もシンプル）
    val tvAppIntent = Intent(Intent.ACTION_VIEW, Uri.parse("vnd.youtube:$videoId"))
        .setPackage("com.google.android.youtube.tv")

    // 2) TV 用 YouTube に watch URL（端末差・バージョン差の保険）
    val tvUrlIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.youtube.com/watch?v=$videoId"))
        .setPackage("com.google.android.youtube.tv")

    // 3) 通常の YouTube アプリ（スマホ・タブレット向け）
    val mobileAppIntent = Intent(Intent.ACTION_VIEW, Uri.parse("vnd.youtube:$videoId"))
        .setPackage("com.google.android.youtube")

    // 4) 最後の手段としてブラウザ
    val browserIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.youtube.com/watch?v=$videoId"))

    val candidates = listOf(tvAppIntent, tvUrlIntent, mobileAppIntent, browserIntent)

    val intentToLaunch = candidates.firstOrNull { it.resolveActivity(pm) != null }
        ?: throw ActivityNotFoundException("No app can handle YouTube video intent")

    // Activity 以外の Context から呼ばれる場合に備える
    intentToLaunch.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)

    startActivity(intentToLaunch)
}
```

### 実装のポイント

* **優先順位付きで Intent を試す**
  Android TV 用 YouTube → URL 指定 → 通常の YouTube アプリ → ブラウザ、の順で安全にフォールバックします。
* **`resolveActivity()` で実行可能かを確認**
  インストールされていないアプリを指定してもクラッシュしないようにしています。
* **`FLAG_ACTIVITY_NEW_TASK` を付与**
  Service や Application Context から呼び出す場合にも対応できます。

## （重要）Android 11 以降のパッケージ可視性対策

Android 11（API 30）以降では、特定のパッケージを `setPackage()` で解決する場合、**Manifest に `queries` の定義が必要になることがあります**。

```xml
<manifest ...>
    <queries>
        <package android:name="com.google.android.youtube.tv" />
        <package android:name="com.google.android.youtube" />
    </queries>
</manifest>
```

### なぜ必要か

* パッケージ可視性制限により、他アプリの存在がデフォルトでは見えなくなるため
* `resolveActivity()` が常に `null` を返してしまう状況を防ぐため

## まとめ

* Android TV では **TV 専用 YouTube アプリのパッケージ名**を明示的に指定するのが重要
* 端末や OS 差を考慮し、**複数の Intent を順に試す設計**が安全
* Android 11 以降では **Manifest の `queries` 設定**を忘れない

この方法を使えば、Android TV 向けアプリから安定して YouTube の特定動画を再生できます。
