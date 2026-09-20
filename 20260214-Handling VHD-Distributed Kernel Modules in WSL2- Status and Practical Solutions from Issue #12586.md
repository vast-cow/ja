---
pubDatetime: 2026-02-14T19:58:22+09:00
title: "WSL2でVHD配布されるKernel Modulesをどう扱うべきか — Issue #12586 から整理する現状と実用解 —"
description: "背景：modules.vhdx 配布への移行、WSL 2.5.1 以降で kernelModules が利用可能、2.5.1 未満でのワークアラウンド（暫定対応）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Microsoft が WSL2 の 6.6 系カーネル以降で **kernel modules を VHD/VHDX として配布する方式**へ移行しました。この変更により、従来の `/lib/modules` 直配置モデルとは異なる運用が必要になります。

本記事では、以下の GitHub Issue をもとに、現在の仕様と実用的な対応策を整理します。

> 参照:
> [https://github.com/microsoft/WSL/issues/12586](https://github.com/microsoft/WSL/issues/12586)

---

## 背景：modules.vhdx 配布への移行

最新の WSL2 カーネル（6.6.y 系）では、カーネルモジュールが `modules.vhdx` として提供されるようになりました。

これにより：

* モジュールが独立した仮想ディスクとして管理される
* カーネル本体（bzImage）とは別に扱われる
* ディストリビューションごとの手動更新が不要になる可能性がある

という構造に変わっています。

しかし、導入当初は設定方法が明確ではなく、Issue #12586 で議論が行われました。

---

# 現在の正式仕様（結論）

## 1. WSL 2.5.1 以降で `kernelModules` が利用可能

`.wslconfig` に以下を設定することで、modules.vhdx を直接指定できます。

```ini
[wsl2]
kernel=C:\\bzImage
kernelModules=C:\\modules.vhdx
```

### 重要条件

* **WSL 2.5.1.0 以上が必要**
* 2.4.x 系では `kernelModules` は boolean 扱いになりエラーになる

エラー例：

```
Invalid boolean value 'C:\modules.vhdx' for key 'wsl2.kernelModules'
```

### アップデート方法

`wsl --update --pre-release` では 2.5.x に上がらないケースがあるため、MSI から手動インストールが必要になる場合があります。

---

## 2. 2.5.1 未満でのワークアラウンド（暫定対応）

正式サポート以前は、以下のような手順が必要でした：

1. `wsl --mount --vhd modules.vhdx`
2. `/mnt/wsl/...` から `/usr/lib/modules` へコピーまたは bind mount
3. systemd で symlink 作成
4. タスクスケジューラで自動マウント

つまり、**かなり運用が煩雑**でした。

2.5.1 以降はこれらは不要になります。

---

# 現在の制限：複数VHDXは指定できない

Issue 内で議論された重要ポイントがこれです。

現在は：

```ini
kernelModules=C:\\modules.vhdx
```

のように **単一 VHDX のみ指定可能**です。

以下は未対応：

```ini
kernelModules=C:\\modules.vhdx;C:\\modules-extra.vhdx   ← 不可
```

---

# なぜ複数VHDXが必要なのか？

議論では次のユースケースが挙がっています。

* USB デバイス専用モジュールを別配布したい
* 組織内カスタムモジュールを追加したい
* out-of-tree モジュールを配布したい
* stock kernel はそのまま使いたい

現在の仕様では：

* modules.vhdx は 1つのみ
* 追加モジュールを別 VHD として積み重ねる仕組みはない
* overlay / FUSE 的な統合機構は未実装

---

# 誤解されがちな点

Issue 内で誤解されていたポイントも整理します。

## ❌ 「modules.vhdx には WSL に必要なモジュールしか入らない」

→ 必要かどうかは利用デバイス次第
→ USB デバイス用のドライバなどは追加が必要なケースがある

## ❌ 「カスタムモジュールを使うならカーネルを再ビルドすべき」

→ 必須ではない

MSI に含まれる upstream kernel の config は公開されています。
そのため：

* stock kernel 用にモジュールだけビルド可能
* out-of-tree module も利用可能

つまり「カーネル再ビルド」は唯一の解ではありません。

---

# 現実的な運用解

現状で実用的なのは次の方法です。

## ✅ 必要なモジュールをまとめた単一 modules.vhdx を生成する

手順の概略：

1. stock kernel 用に module をビルド
2. `gen_modules_vhdx.sh` で modules.vhdx を生成
3. `.wslconfig` で指定

これが最も安定した方法です。

---

# 今後の改善余地

Issue で要望されているのは：

* `kernelModules` に複数 VHDX 指定対応
* `/lib/modules/$(uname -r)/extra/volX` などへの分離マウント
* depmod による動的統合

現時点では未実装ですが、ユースケースとしては妥当です。

---

# まとめ

| 項目                           | 状況                 |
| ---------------------------- | ------------------ |
| modules.vhdx 正式サポート          | WSL 2.5.1 以降       |
| `.wslconfig` 設定              | `kernelModules=パス` |
| 複数 VHDX 指定                   | 未対応                |
| stock kernel + custom module | 可能                 |
| 推奨運用                         | 単一 VHDX に統合        |

---

## 結論

* WSL 2.5.1 以降で `kernelModules` が正式利用可能
* 現在は単一 VHDX のみ対応
* custom module を使う場合は、必要なものをまとめた modules.vhdx を作るのが現実解

詳細な議論は以下を参照してください：

[https://github.com/microsoft/WSL/issues/12586](https://github.com/microsoft/WSL/issues/12586)
