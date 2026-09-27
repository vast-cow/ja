---
pubDatetime: 2026-02-14T17:33:19+09:00
title: "Windows 11 Modern Standby中にネットワークを切断する設定"
description: "グループポリシーで「スタンバイ中のネットワーク接続」を無効化（Pro/Enterprise向け）、Modern Standby（S0）かどうかを確認する、注意点（副作用）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Windows 11 の **Modern Standby（S0 低電力アイドル）** は、スリープ中でも通知や同期のためにネットワークが維持されることがあります。

「スリープ中はネットワークを切ってほしい」「夜間のバックグラウンド通信を止めたい」といった用途では、**Modern Standby 中のネットワーク接続を許可しない設定**が有効です。

この記事では、Windows 11 Pro / Enterprise に加えて、**Windows 11 Home Edition での設定方法**も解説します。

## この記事でやること

- Modern Standby（S0）中の **ネットワーク接続を許可しない**
- 電源接続時／バッテリ使用時の両方を制御する
- Windows 11 Home Edition ではレジストリから設定する
- 設定後に Modern Standby 対応PCであることを確認する

## グループポリシーで「スタンバイ中のネットワーク接続」を無効化（Pro / Enterprise向け）

Windows 11 Pro / Enterprise では、**ローカル グループ ポリシー エディター**から設定できます。

### 手順

1. `Win + R` を押して **ファイル名を指定して実行**を開く
2. `gpedit.msc` を入力して **ローカル グループ ポリシー エディター**を起動
3. 次の場所へ移動する

**コンピューターの構成**  
→ **管理用テンプレート**  
→ **システム**  
→ **電源の管理**  
→ **スリープの設定**

4. 以下のポリシーを探して、どちらも **「無効」** に設定する

- **コネクト スタンバイ中にネットワーク接続を許可する (電源接続時)**
  - 英語表記：Allow network connectivity during connected-standby (plugged in)
- **コネクト スタンバイ中にネットワーク接続を許可する (バッテリ使用時)**
  - 英語表記：Allow network connectivity during connected-standby (on battery)

5. 設定を反映する

管理者としてコマンドプロンプトを開き、以下を実行します。

```bat
gpupdate /force
```

または、Windowsを再起動します。

---

## Windows 11 Home Editionで設定する方法

Windows 11 Home Edition には通常 `gpedit.msc` が搭載されていないため、同等の設定を**レジストリから直接指定**します。

### 方法1：コマンドで設定する

**Windows ターミナル（管理者）**または**コマンドプロンプト（管理者）**を開き、以下の2つのコマンドを実行します。

```bat
reg add "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v ACSettingIndex /t REG_DWORD /d 0 /f

reg add "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v DCSettingIndex /t REG_DWORD /d 0 /f
```

設定後、Windowsを**再起動**します。

それぞれの意味は以下のとおりです。

- `ACSettingIndex = 0`
  - 電源接続時に Modern Standby 中のネットワーク接続を許可しない
- `DCSettingIndex = 0`
  - バッテリ使用時に Modern Standby 中のネットワーク接続を許可しない

つまり、Pro / Enterprise のグループポリシーで「電源接続時」「バッテリ使用時」の両方を無効にする設定に相当します。

### 方法2：レジストリエディターから設定する

GUIで設定したい場合は、`regedit` を使用します。

1. `Win + R` を押す
2. `regedit` と入力して **レジストリエディター**を起動
3. 次のキーへ移動する

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9
```

途中の `Power`、`PowerSettings`、GUIDのキーが存在しない場合は、新しく作成します。

4. GUIDキーの中に以下の **DWORD (32ビット) 値**を作成する

```text
ACSettingIndex
DCSettingIndex
```

5. 両方の値を `0` に設定する
6. Windowsを再起動する

### Home Editionで設定を元に戻す

ポリシーによる強制を解除したい場合は、作成した値を削除します。

管理者のコマンドプロンプトで以下を実行します。

```bat
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v ACSettingIndex /f

reg delete "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v DCSettingIndex /f
```

その後、Windowsを再起動します。

---

## Modern Standby（S0）かどうかを確認する

設定の前提として、PCが Modern Standby を採用しているか確認します。

コマンドプロンプトまたはWindows ターミナルで以下を実行します。

```bat
powercfg /a
```

出力の「このシステムで利用可能なスリープ状態」に、

```text
スタンバイ (S0 低電力アイドル)
```

が含まれていれば、そのPCは Modern Standby に対応しています。

環境によっては、

```text
スタンバイ (S0 低電力アイドル) ネットワーク接続あり
```

またはネットワーク関連の異なる表記になる場合があります。

## 設定後の動作を確認する

Modern Standby はPCメーカー、NIC、ドライバー、ファームウェアなどの影響も受けるため、レジストリやポリシーを設定しただけで「物理的にNICが完全停止する」とは限りません。

実際にスリープさせたあと、必要に応じて以下のコマンドで Modern Standby 中の動作を確認できます。

```bat
powercfg /sleepstudy
```

実行すると **SleepStudy レポート**が生成されます。

スリープ中に意図しないネットワーク通信やバックグラウンド動作が続いていないか確認する際に利用できます。

## 注意点（副作用）

- スリープ中の **通知・同期・メール受信・バックグラウンド通信**などが停止または制限される可能性があります
- PCメーカーやNIC、ドライバー、ファームウェアの実装によって挙動が異なる場合があります
- Modern Standbyそのものを無効化して従来のS3スリープへ変更する設定ではありません
- レジストリを変更する場合は、値やキーを間違えないよう注意してください

## まとめ

Windows 11 の Modern Standby 中にネットワーク通信を抑制したい場合、Windowsのエディションによって設定方法が異なります。

**Windows 11 Pro / Enterprise**

グループポリシーで以下の2項目を **「無効」** にします。

- **コネクト スタンバイ中にネットワーク接続を許可する (電源接続時)**
- **コネクト スタンバイ中にネットワーク接続を許可する (バッテリ使用時)**

**Windows 11 Home Edition**

グループポリシーエディターが標準搭載されていないため、以下のレジストリポリシーを使用します。

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9
```

ここに、

```text
ACSettingIndex = 0
DCSettingIndex = 0
```

を設定します。

これにより、電源接続時・バッテリ使用時の両方について、Modern Standby 中のネットワーク接続を許可しない設定にできます。
