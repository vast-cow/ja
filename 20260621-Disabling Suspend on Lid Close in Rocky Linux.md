---
pubDatetime: 2026-06-21T15:50:22+09:00
title: "Rocky LinuxでLid Close時のサスペンドを無効化する方法"
description: "Rocky Linux では、multi-user.target でも蓋閉じ処理は基本的に systemd-logind が扱います。HandleLidSwitch=ignore にします。Rocky 公式ドキュメントでも /etc/systemd/logind.conf の HandleLidSw…"
---

Rocky Linux では、`multi-user.target` でも蓋閉じ処理は基本的に **systemd-logind** が扱います。`HandleLidSwitch=ignore` にします。Rocky 公式ドキュメントでも `/etc/systemd/logind.conf` の `HandleLidSwitch` を `ignore` にする方法が案内されています。([Rocky Linux Docs][1])

## 推奨設定

```bash
sudo cp -a /etc/systemd/logind.conf /etc/systemd/logind.conf.bak.$(date +%F-%H%M%S)

sudo mkdir -p /etc/systemd/logind.conf.d

sudo tee /etc/systemd/logind.conf.d/99-ignore-lid.conf >/dev/null <<'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF

sudo systemctl restart systemd-logind.service
```

これで、通常時・AC 接続時・ドック接続時の蓋閉じをすべて無視します。`HandleLidSwitch` の既定値は `suspend` で、`ignore` にすると logind が蓋閉じイベントでサスペンドしなくなります。([man7.org][2])

## 確認

```bash
systemd-analyze cat-config systemd/logind.conf | grep -E 'HandleLidSwitch'
```

期待値:

```text
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

現在の default target も確認するなら:

```bash
systemctl get-default
```

`multi-user.target` にしておくなら:

```bash
sudo systemctl set-default multi-user.target
```

## 直接 `/etc/systemd/logind.conf` を編集する場合

drop-in が効かない環境では、直接編集でも可です。

```bash
sudo vi /etc/systemd/logind.conf
```

以下を `[Login]` セクションに設定します。

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

反映:

```bash
sudo systemctl restart systemd-logind.service
```

## まだスリープする場合

まずログを見ます。

```bash
journalctl -u systemd-logind -b | grep -i -E 'lid|suspend|sleep'
```

さらに、他のプロセスが電源管理を握っていないか確認します。

```bash
systemd-inhibit --list
```

`multi-user.target` なら通常はデスクトップ環境が介入しませんが、GNOME などの graphical session が動いている場合は、デスクトップ環境側がサスペンド処理を引き取ることがあります。systemd の man page でも、別アプリケーションが低レベル inhibitor lock を取ると `Handle*` 設定が効かない場合があると説明されています。([man7.org][2])

強制的にサスペンド系 target 自体を無効化する最終手段はこれです。

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

戻す場合:

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

通常は **logind の `HandleLidSwitch=ignore` だけで足ります**。サーバー用途のノート PC なら、蓋を閉じたまま高負荷運用すると排熱が悪化する機種がある点だけ注意してください。Red Hat の手順にも同様の注意があります。([Red Hat Documentation][3])

[1]: https://docs.rockylinux.org/10/gemstones/scripts/NoSleep/ "NoSleep.sh - A simple Configuration Script - Documentation"
[2]: https://man7.org/linux/man-pages/man5/logind.conf.5.html "logind.conf(5) - Linux manual page"
[3]: https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/7/html/desktop_migration_and_administration_guide/closing-lid "13.10. ノート PC を閉じた際にコンピューターがサスペンドしないようにする | デスクトップの移行および管理ガイド | Red Hat Enterprise Linux | 7 | Red Hat Documentation"
