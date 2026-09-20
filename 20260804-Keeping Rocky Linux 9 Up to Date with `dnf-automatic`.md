---
title: "`dnf-automatic` を使用して Rocky Linux 9 を最新の状態に保つ"
description: ""
pubDatetime: 2026-08-04T06:50:08.333Z
updatedDate: 2026-08-04T06:51:09.476Z
---

システムを最新の状態に維持することは、セキュリティ、安定性、およびパフォーマンスを確保するために最も重要な作業の一つです。**Rocky Linux 9.6** では、`dnf-automatic` パッケージを使用することで、利用可能な更新を定期的に確認し、管理者への通知、パッケージのダウンロード、または更新の自動インストールを実行できます。

このガイドでは、インストール、設定、自動更新の構成、およびパッケージのダウンロードやインストールを行わずに MOTD を使用して通知のみを行う構成について説明します。

---

## 1. `dnf-automatic` をインストールする

`dnf-automatic` はデフォルトではインストールされていません。次のコマンドでインストールします。

```bash
sudo dnf install -y dnf-automatic
```

---

## 2. `dnf-automatic` を設定する

メインの設定ファイルは次の場所にあります。

```text
/etc/dnf/automatic.conf
```

お好みのテキストエディタで開きます。

```bash
sudo nano /etc/dnf/automatic.conf
```

### 主な設定項目

#### 更新の種類

`upgrade_type` は、どの種類の更新を検出するかを指定します。

```ini
[commands]
upgrade_type = security
```

指定できる値は次のとおりです。

* `default` — 利用可能なすべての更新
* `security` — セキュリティアドバイザリに関連付けられた更新のみ

#### 更新パッケージのダウンロード

パッケージを自動的にダウンロードする場合は、次のように設定します。

```ini
download_updates = yes
```

自動ダウンロードを無効にする場合は、次のように設定します。

```ini
download_updates = no
```

#### 更新の自動適用

利用可能な更新を自動的にインストールする場合は、次のように設定します。

```ini
apply_updates = yes
```

自動インストールを無効にする場合は、次のように設定します。

```ini
apply_updates = no
```

`apply_updates = yes` を設定する場合は、パッケージのダウンロードも有効になっている必要があります。

#### Emitters（通知方法）

`[emitters]` セクションでは、結果の通知方法を指定します。利用できる通知方法には、標準出力、メール、カスタムコマンド、および MOTD があります。

たとえば、MOTD を使用して通知する場合は次のように設定します。

```ini
[emitters]
emit_via = motd
```

MOTD エミッターは更新結果を `/etc/motd` に書き込み、SSH やローカルコンソールでログインした際に管理者が確認できるようにします。

---

## 3. 自動更新を有効にする

汎用の `dnf-automatic.timer` は、`/etc/dnf/automatic.conf` に設定された内容に従って動作します。

たとえば、セキュリティ更新を自動的にダウンロードしてインストールする場合は、次のように設定します。

```ini
[commands]
upgrade_type = security
download_updates = yes
apply_updates = yes
```

その後、タイマーを有効化して起動します。

```bash
sudo systemctl enable --now dnf-automatic.timer
```

これにより、`dnf-automatic` は設定ファイルに従って更新を確認し、必要な処理を実行します。

---

## 4. MOTD を使用した通知のみの構成

パッケージをダウンロードまたはインストールせず、利用可能な更新だけを管理者へ通知する場合は、`dnf-automatic` を MOTD エミッターで構成します。

`/etc/dnf/automatic.conf` を編集します。

```bash
sudo nano /etc/dnf/automatic.conf
```

次のように設定します。

```ini
[commands]
upgrade_type = security
download_updates = no
apply_updates = no

[emitters]
emit_via = motd
```

この構成では次の動作を行います。

* 利用可能なセキュリティ更新を確認する
* 更新パッケージはダウンロードしない
* 更新パッケージはインストールしない
* 結果を `/etc/motd` に書き込む

通知専用タイマーを有効にします。

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

`dnf-automatic-notifyonly.timer` は、ダウンロードおよびインストール動作を上書きし、通知のみを実行します。`download_updates = no` および `apply_updates = no` を設定ファイルに明示的に記述しておくことで、設定内容が分かりやすくなり、汎用タイマーを使用した場合でも安全なデフォルトとして機能します。

### ダウンロードおよびインストール用タイマーを無効にする

他の `dnf-automatic` タイマーによってパッケージがダウンロードまたはインストールされないようにするため、汎用タイマー、ダウンロードタイマー、およびインストールタイマーを無効にします。

```bash
sudo systemctl disable --now dnf-automatic.timer
sudo systemctl disable --now dnf-automatic-download.timer
sudo systemctl disable --now dnf-automatic-install.timer
```

その後、通知専用タイマーのみを有効にします。

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

有効になっているタイマーを確認します。

```bash
systemctl list-timers --all | grep dnf-automatic
```

通知専用構成では、`dnf-automatic-notifyonly.timer` のみが有効になっていることを確認してください。

### MOTD の出力を確認する

設定をテストするためにサービスを手動で実行します。

```bash
sudo systemctl start dnf-automatic-notifyonly.service
```

生成された MOTD を表示します。

```bash
cat /etc/motd
```

システムのログイン設定で `/etc/motd` が表示されるようになっていれば、次回 SSH またはローカルコンソールでログインした際にも同じ内容が表示されます。

> **注:** 更新の確認では、リポジトリメタデータの更新または読み込みが必要です。通知専用構成では RPM パッケージのダウンロードは行われませんが、リポジトリメタデータの通信自体は発生します。

---

## 5. その他のタイマーモード

`dnf-automatic` には、用途に応じた複数の systemd タイマーが用意されています。

### `/etc/dnf/automatic.conf` の設定に従う

```bash
sudo systemctl enable --now dnf-automatic.timer
```

### 通知のみ

更新を確認し、ダウンロードやインストールを行わずに結果だけを通知します。

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

### ダウンロードのみ

利用可能な更新パッケージをダウンロードしますが、インストールは行いません。

```bash
sudo systemctl enable --now dnf-automatic-download.timer
```

### ダウンロードとインストール

更新パッケージをダウンロードし、そのままインストールします。

```bash
sudo systemctl enable --now dnf-automatic-install.timer
```

これらの専用タイマーは、`/etc/dnf/automatic.conf` に設定されたダウンロードおよびインストールに関する項目を必要に応じて上書きして動作します。

---

## 6. 動作確認とトラブルシューティング

通知専用タイマーの状態を確認します。

```bash
systemctl status dnf-automatic-notifyonly.timer
```

次回の実行予定を確認します。

```bash
systemctl list-timers --all | grep dnf-automatic
```

サービスログを確認します。

```bash
journalctl -u dnf-automatic-notifyonly.service
```

`dnf-automatic` に関連するすべてのログを確認します。

```bash
journalctl -u 'dnf-automatic*'
```

各タイマーが有効かどうかを確認します。

```bash
systemctl is-enabled dnf-automatic.timer
systemctl is-enabled dnf-automatic-notifyonly.timer
systemctl is-enabled dnf-automatic-download.timer
systemctl is-enabled dnf-automatic-install.timer
```

通知専用構成では、期待される結果は次のとおりです。

```text
disabled
enabled
disabled
disabled
```

---

## まとめ

Rocky Linux 9.6 では、`dnf-automatic` を次のような用途に応じて利用できます。

* 手動操作なしで常に最新の状態を維持したいシステム向けの自動インストール
* インストールは別途実施し、事前にパッケージだけを取得したい環境向けの自動ダウンロード
* 更新内容を管理者が確認したうえで適用する運用向けの通知のみ

パッケージをダウンロードせず、通知のみを行う構成は次のとおりです。

```ini
[commands]
upgrade_type = security
download_updates = no
apply_updates = no

[emitters]
emit_via = motd
```

有効にするタイマーは次のものだけです。

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

この構成では、利用可能なセキュリティ更新が MOTD を通じて通知され、パッケージのダウンロードおよびインストールは手動管理のまま維持されます。
