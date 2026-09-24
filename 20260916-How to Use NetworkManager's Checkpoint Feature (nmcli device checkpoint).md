---
title: "NetworkManagerのチェックポイント機能（nmcli device checkpoint）の使い方"
description: "`nmcli device checkpoint` は、**NetworkManager の設定変更を安全に試すための「復元ポイント（チェックポイント）」機能**です。"
pubDatetime: 2026-09-16T04:15:12.719Z
---

`nmcli device checkpoint` は、**NetworkManager の設定変更を安全に試すための「復元ポイント（チェックポイント）」機能**です。

リモートサーバー（SSH接続中など）でネットワーク設定を変更する際に、設定ミスで通信が切断されても、自動的に元の状態へ戻せるため非常に便利です。

---

# 基本的な仕組み

通常、ネットワーク設定を変更すると、

```plaintext
現在の設定
      │
      ▼
新しい設定を適用
      │
      ├──成功 → そのまま使う
      │
      └──失敗 → 通信断
```

となります。

Checkpointを使うと、

```plaintext
Checkpoint作成
      │
      ▼
新しい設定を適用
      │
      ├──成功
      │      │
      │      ▼
      │  Checkpointを破棄
      │
      └──失敗
             │
             ▼
      タイムアウト後に自動ロールバック
```

という流れになります。

---

# 基本構文

```bash
nmcli device checkpoint create [DEVICE...] --timeout 秒
```

例

```bash
nmcli device checkpoint create eth0 --timeout 60
```

これは

* eth0 の状態を保存
* 60秒以内に確定しなければ元へ戻す

という意味です。

---

# 実際の利用例

例えばIPアドレスを変更したい場合

```bash
nmcli device checkpoint create eth0 --timeout 60
```

Checkpointが作成されます。

続いて

```bash
nmcli con modify eth0 \
    ipv4.addresses 192.168.1.50/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.method manual

nmcli con up eth0
```

を実行します。

もしSSHが切れてしまっても、

```plaintext
60秒後
```

NetworkManager が自動で

```plaintext
元のIP
元のGateway
元のRoute
```

へ戻します。

---

# 成功したら確定する

変更が成功し通信できることを確認したら

```bash
nmcli device checkpoint destroy <checkpoint-id>
```

を実行します。

すると

```plaintext
チェックポイント削除
＝ロールバックしない
```

となります。

---

# 手動でロールバック

途中で失敗したと判断したら

```bash
nmcli device checkpoint rollback <checkpoint-id>
```

で即座に元へ戻せます。

---

# チェックポイント一覧

現在のCheckpointは

```bash
nmcli device checkpoint show
```

で確認できます。

例

```plaintext
ID   CREATED              TIMEOUT
3    2025-01-01 10:00     60
```

---

# 複数デバイスも可能

例えば

```bash
nmcli device checkpoint create eth0 bond0 br0 --timeout 120
```

のようにすると

```plaintext
eth0
bond0
br0
```

すべてまとめて保存されます。

ブリッジやボンディング構成ではよく利用されます。

---

# `--timeout` の意味

例えば

```bash
--timeout 30
```

なら

```plaintext
Checkpoint作成
        │
0秒     │
10秒    │ 設定変更
20秒    │ SSH確認
30秒    │ destroyしなければ自動復元
```

となります。

---

# リモート運用での典型例

SSHで接続中にIP変更を試す場合は、次のような手順が安全です。

```bash
nmcli device checkpoint create eth0 --timeout 120

nmcli con modify ...

nmcli con up ...

# SSHが継続していることを確認

nmcli device checkpoint destroy <id>
```

もし接続できなくなれば、120秒後に自動で元へ戻るため、現地での復旧作業が不要になることがあります。

---

# 注意点

* **NetworkManagerが管理しているデバイスのみ**が対象です。
* ロールバックできるのは、**NetworkManager が変更したネットワーク状態**です。NetworkManagerの管理外で行った変更（例えば `ip` コマンドで直接追加した設定など）は対象外です。
* タイムアウト後の自動復元は、NetworkManagerサービスが動作し続けていることが前提です。

---

## まとめ

`nmcli device checkpoint` は、ネットワーク設定変更時の「セーフティネット」と考えると分かりやすい機能です。

* `create`：現在のネットワーク状態を保存
* `show`：チェックポイント一覧を表示
* `destroy`：変更を確定（ロールバックしない）
* `rollback`：即座に保存時点の状態へ戻す
* `--timeout`：指定時間内に `destroy` されなければ自動でロールバック

SSH経由でIPアドレス、ルーティング、ブリッジ、VLANなどを変更する運用では、設定ミスによる通信断への保険として非常に有用です。
