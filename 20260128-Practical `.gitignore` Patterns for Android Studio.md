---
pubDatetime: 2026-01-28T23:22:10+09:00
title: "Android Studio 向け `.gitignore` の考え方と実務での定番構成"
description: "Android Studio 向け .gitignore の考え方と実務での定番構成 Android Studio（Gradle / Kotlin ベース）で Android アプリを開発する際、.gitignore をどう書くかは多くの開発者が一度は悩むポイントです。実際には、ネット上の議論や実務…"
---

# Android Studio 向け `.gitignore` の考え方と実務での定番構成

Android Studio（Gradle / Kotlin ベース）で Android アプリを開発する際、`.gitignore` をどう書くかは多くの開発者が一度は悩むポイントです。実際には、ネット上の議論や実務の知見はある程度収束しており、「何を Git 管理から外すか」という考え方はかなり共通しています。

この記事では、広く参照されている公式テンプレートや実務上の論点を整理し、**まず採用しやすい無難な `.gitignore`** と、**意見が分かれやすいポイント**を簡潔にまとめます。



## Android 向け `.gitignore` の基本方針

Android Studio 向けの `.gitignore` は、主に次の4カテゴリを除外する方針で構成されます。

* **ビルド成果物・キャッシュ**
  再生成可能で、差分がノイズになりやすいもの
* **端末・環境依存の設定**
  SDK パスなど、他人の環境では意味を持たないもの
* **IDE の生成物・ユーザー固有設定**
  Android Studio / IntelliJ が自動生成する設定
* **秘匿情報（特に署名関連）**
  漏洩すると事故につながるもの

この考え方の出発点として、多くの人が参照しているのが GitHub 公式の `Android.gitignore` です。
Stack Overflow や日本語記事でも「まずはこれをベースにする」という扱いが一般的です。
([GitHub][1])



## 基準としてよく参照される公式情報

### GitHub の `Android.gitignore`

GitHub が公開している `Android.gitignore` には、次のような項目が含まれています（要旨）。

* Gradle 関連: `.gradle/`, `build/`
* 環境依存: `local.properties`
* Android Studio 生成物: `captures/`, `.externalNativeBuild/`, `.cxx/`
* 出力物: `*.apk`, `*.aab`, `output-metadata.json`
* IntelliJ / Android Studio: `*.iml`, `.idea/`
* 署名鍵: `*.jks`, `*.keystore`
* Firebase / Google: `google-services.json`
* プロファイル: `*.hprof`

これは GitHub の `gitignore` リポジトリ全体の趣旨とも一致しており、
「公式テンプレをベースにする」という姿勢自体が広く支持されています。
([GitHub][2])



### JetBrains（IntelliJ / Android Studio）の公式見解

一方で、**`.idea` ディレクトリをどう扱うか**については意見が分かれます。

JetBrains の公式ガイド（2025-11-26 更新）では、次のように整理されています。

* `.idea` 配下は **基本的に共有（コミット）してよい**
* ただし、`workspace.xml` や `shelf/` などの **ユーザー固有ファイルは除外**
* Gradle / Maven プロジェクトでは、`*.iml` や一部設定ファイルは
  「生成物なので共有しない判断もあり得る」

つまり JetBrains 的には「`.idea` を丸ごと ignore するのは必ずしも正解ではない」が、
Gradle ベースの Android プロジェクトでは、実務的な柔軟性も認めています。
([JetBrains][3])



### Kotlin を使う場合の注意点

Kotlin Gradle Plugin は、プロジェクト直下に `.kotlin/` ディレクトリを生成します。
これは公式ドキュメントで **「コミットしないこと」** が明言されています。

そのため、Kotlin を使うプロジェクトでは `.kotlin/` を ignore に追加するのが定番です。
([Kotlin][4])



## 意見が分かれるポイント①：`.idea/` をどうするか

実運用では、次の2つの考え方に大別されます。

### A) `.idea/` を丸ごと ignore する

* GitHub の `Android.gitignore` はこの立場
* 古い AOSP のデモプロジェクトでも同様の例あり
* **メリット**

  * IDE 設定差分のノイズが減る
  * チームで IDE やプラグイン差があっても揉めにくい
* **デメリット**

  * run configuration や code style を共有しづらい

「プロジェクト構造の正は Gradle にあり、IDE 設定は個人差が大きい」という考え方の人は、
このシンプルな運用を選ぶことが多いです。
([Qiita][6])



### B) `.idea/` は共有できるものだけ残す

* JetBrains 公式はこの方針
* GitHub の `Global/JetBrains.gitignore` も部分的除外の設計
* **メリット**

  * code style や run config をチームで揃えやすい
* **デメリット**

  * プラグイン差などで `.idea` が汚れやすい

Android Studio をチーム標準 IDE として運用する場合には、
この折衷案が選ばれることもあります。
([GitHub][7])



## 意見が分かれるポイント②：`google-services.json`

`google-services.json` は秘密鍵そのものではありませんが、扱いは方針次第です。

* **コミットしてよい**という立場

  * APK に含まれる情報で、解析すれば取得可能
  * API key には利用制限をかけるのが前提
    ([Google Groups][8], [Stack Overflow][9])

* **public / OSS では ignore する**という立場

  * リポジトリにはサンプルファイルのみ置く
  * 公開範囲に応じた判断が必要
    ([Reddit][10], [droidcon][11])

GitHub の `Android.gitignore` は、デフォルトでは ignore に含めています。
([GitHub][2])



## 実務で無難なおすすめ `.gitignore`

以下は、

* GitHub の `Android.gitignore`
* Kotlin 利用時の公式注意点

を踏まえた、**初手として採用しやすい構成**です。

```gitignore
# ===== OS / editor =====
.DS_Store
Thumbs.db
*.swp
*.swo

# ===== Gradle / Kotlin =====
.gradle/
.kotlin/
build/
local.properties

# ===== Android Studio / IntelliJ =====
*.iml
.idea/
*.ipr
*.iws
out/

# ===== Android build outputs =====
*.apk
*.aab
output-metadata.json

# ===== Android Studio generated =====
captures/
.externalNativeBuild/
.cxx/

# ===== Profiling =====
*.hprof

# ===== Signing / secrets =====
*.jks
*.keystore
keystore.properties
signing.properties

# ===== Google Services (方針で決める) =====
# google-services.json
```

この構成は「差分が荒れにくく、チーム内で揉めにくい」という理由から、
多くの現場で無難な選択とされています。
([GitHub][2])



## `.idea/` を共有したい場合の落としどころ

`.idea` を完全に無視せず、ユーザー固有・生成物だけを除外する方法もあります。

```gitignore
.idea/**/workspace.xml
.idea/**/usage.statistics.xml
.idea/**/shelf/

.idea/**/tasks.xml
.idea/**/dictionaries/

.idea/**/modules.xml
.idea/**/gradle.xml
.idea/**/libraries/

*.iml
```

JetBrains の公式方針と、Gradle プロジェクトの実情を組み合わせた現実的な折衷案です。
([JetBrains][3])



## `.gitignore` を生成する便利な方法

OS やツールをまとめて指定したい場合は、gitignore.io（Toptal 提供）が便利です。

```bash
curl -L "https://www.toptal.com/developers/gitignore/api/android,androidstudio,gradle,kotlin,macos,windows,linux" > .gitignore
```

([Toptal][12])



## ありがちな注意点

* `.gitignore` に書いても、**すでに追跡中のファイルは消えません**
  → `git rm --cached` が必要
  ([Stack Overflow][13])
* `google-services.json` は

  * チーム内共有が必要 → コミット
  * public / OSS → ignore + サンプル
    という分岐が現実的です。
    ([Google Groups][8])


[1]: https://github.com/github/gitignore
[2]: https://raw.githubusercontent.com/github/gitignore/master/Android.gitignore
[3]: https://intellij-support.jetbrains.com/hc/en-us/articles/206544839-How-to-manage-projects-under-Version-Control-Systems
[4]: https://kotlinlang.org/docs/gradle-configure-project.html
[6]: https://qiita.com/blue928sky/items/d98153a0964301aced74
[7]: https://raw.githubusercontent.com/github/gitignore/master/Global/JetBrains.gitignore
[8]: https://groups.google.com/g/firebase-talk/c/bamCgTDajkw/m/uVEJXjtiBwAJ%29
[9]: https://stackoverflow.com/questions/37358340/should-i-add-the-google-services-json-from-firebase-to-my-repository
[10]: https://www.reddit.com/r/androiddev/comments/ucscrj/should_i_commit_my_googleservicesjson_into_my/
[11]: https://www.droidcon.com/2022/11/15/google-services-json-file-to-commit-or-not-to-commit-thats-the-question/
[12]: https://www.toptal.com/developers/gitignore
[13]: https://stackoverflow.com/questions/62349212/how-should-i-hide-my-googleservices-json-file-using-gitignore
