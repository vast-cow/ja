---
pubDatetime: 2026-07-01T13:41:29+09:00
title: "Rocky Linuxのバージョン更新"
description: "現在のバージョン確認、パッケージ情報の更新、システム全体を更新を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Rocky Linux 9.5 から 9.6 へのアップデートは通常の DNF 更新で可能です。

## 現在のバージョン確認

```bash
cat /etc/os-release
```

または

```bash
cat /etc/rocky-release
```

## パッケージ情報の更新

まずリポジトリ情報を更新します。

```bash
sudo dnf clean all
sudo dnf makecache
```

## システム全体を更新

```bash
sudo dnf upgrade --refresh
```

または

```bash
sudo dnf update --refresh
```

Rocky Linux はマイナーリリース（9.5 → 9.6）を固定せず、利用可能な最新の 9.x に更新されます。

## 再起動

カーネルや systemd が更新された場合は再起動します。

```bash
sudo reboot
```

## バージョン確認

再起動後に確認します。

```bash
cat /etc/rocky-release
```

期待される出力例:

```text
Rocky Linux release 9.6 (Blue Onyx)
```

---

## ELevateやLeappは不要

9.5 → 9.6 は同一メジャーバージョン内のアップデートなので、

* Leapp
* ELevate

は不要です。

これらは 8.x → 9.x や 9.x → 10.x のようなメジャーアップグレードで使用します。

---

## 特定のマイナーバージョンに固定している場合

以下で確認できます。

```bash
sudo dnf config-manager --dump | grep releasever
```

または

```bash
cat /etc/dnf/vars/releasever
```

`9.5` が設定されている場合は削除するか `9` に変更します。

```bash
sudo rm -f /etc/dnf/vars/releasever
```

その後、

```bash
sudo dnf upgrade --refresh
```

を実行してください。

