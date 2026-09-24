---
pubDatetime: 2026-04-22T23:46:12+09:00
title: "Raspberry Pi 4 + Raspberry Pi OS + Docker で Home Assistant と Eufy IndoorCam 2K Pan & Tilt を連携し、ペット検出時に画像保存する"
description: "この記事でやりたいこと、まず結論、Raspberry Pi OS を準備するを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

# Raspberry Pi 4 + Raspberry Pi OS + Docker で Home Assistant と Eufy IndoorCam 2K Pan & Tilt を連携し、ペット検出時に画像保存する

Anker の **Eufy IndoorCam 2K Pan & Tilt** を Home Assistant と連携し、**ペットを検出したタイミングで画像を保存したい**。
今回はその構成を、**Raspberry Pi 4 / Raspberry Pi OS / Docker** 環境で組む手順としてまとめる。

この記事では、まず実際に動かすためのセットアップ手順を先に説明し、そのあとで途中で出た疑問点や詰まりポイントを Q&A として整理する。

---

## この記事でやりたいこと

やることはシンプルで、次の流れを作る。

1. Raspberry Pi 4 上で Home Assistant を Docker で動かす
2. Eufy 用の websocket ブリッジを別コンテナで動かす
3. Home Assistant に Eufy Security カスタム統合を入れる
4. ペット検出時に Event Image を保存する automation を作る

最終的な構成イメージはこうなる。

```text
Raspberry Pi OS 64-bit
  └─ Docker / Docker Compose
      ├─ Home Assistant Container
      └─ eufy-security-ws

Home Assistant
  ├─ HACS
  ├─ Eufy Security カスタム統合
  └─ automation
       petDetected → Event Image を snapshot 保存
```

---

## まず結論

**Raspberry Pi 4 + Raspberry Pi OS + Docker で構築可能。**

ただし、ポイントは次の通り。

* **Raspberry Pi OS 64-bit** を使う
* Home Assistant は **Container 運用** にする
* Eufy 連携は **標準統合ではなく `eufy_security` カスタム統合** を使う
* websocket ブリッジとして **`eufy-security-ws` を別コンテナで動かす**
* ペット検出時の画像保存は、**`Event Image` を `image.snapshot` で保存**する

---

# 手順解説

## 1. Raspberry Pi OS を準備する

まず、Raspberry Pi 4 に **Raspberry Pi OS 64-bit** を入れておく。
32-bit ではなく 64-bit を推奨する。Docker や今後の運用を考えるとこちらのほうが無難。

OS 更新もしておく。

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

再起動後、64-bit か確認する。

```bash
uname -m
```

`aarch64` なら 64-bit。

---

## 2. Docker をインストールする

Docker と Docker Compose を使える状態にする。

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

---

## 3. 作業ディレクトリを作る

Home Assistant と Eufy 関連のデータを置くディレクトリを作る。

```bash
mkdir -p ~/ha-stack
cd ~/ha-stack

mkdir -p homeassistant/config
mkdir -p homeassistant/media/eufy_snapshots
mkdir -p eufy-security-ws
```

---

## 4. `compose.yaml` を作る

この構成では、Home Assistant と `eufy-security-ws` を別コンテナで起動する。

`~/ha-stack/compose.yaml`

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Asia/Tokyo
    volumes:
      - ./homeassistant/config:/config
      - ./homeassistant/media:/media
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro

  eufy-security-ws:
    container_name: eufy-security-ws
    image: bropat/eufy-security-ws:latest
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: Asia/Tokyo
      USERNAME: 'your-eufy-mail@example.com'
      PASSWORD: 'your-password'
      COUNTRY: 'JP'
      TRUSTED_DEVICE_NAME: 'rpi4-ha'
      EVENT_DURATION_SECONDS: '10'
    volumes:
      - ./eufy-security-ws:/data
```

起動する。

```bash
docker compose up -d
```

Home Assistant には `http://<PiのIP>:8123` でアクセスできる。

---

## 5. `configuration.yaml` に保存先を追加する

ペット検出時の画像を `/media/eufy_snapshots` に保存するので、そのディレクトリを Home Assistant に許可する。

`~/ha-stack/homeassistant/config/configuration.yaml` に追記する。

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

ここで重要なのは、**既存ファイルがあれば上書きではなく追記**すること。
すでに `homeassistant:` ブロックがあるなら、その中に `allowlist_external_dirs` を追加する。

変更後は再起動する。

```bash
docker compose restart homeassistant
```

---

## 6. HACS をインストールする

`eufy_security` は標準統合ではないため、**HACS** が必要。

Home Assistant コンテナ内で HACS を導入する。

```bash
docker exec -it homeassistant bash
wget -O - https://get.hacs.xyz | bash -
exit
```

その後、Home Assistant を再起動する。

```bash
docker compose restart homeassistant
```

再起動後、Home Assistant UI で HACS を追加する。

* **Settings → Devices & Services**
* **Add Integration**
* **HACS**
* GitHub device login を完了

---

## 7. `eufy-security-ws` のログイン情報を設定する

ここはかなり重要。
もし `docker logs -f eufy-security-ws` で次のように出たら、

```text
Missing one of USERNAME or PASSWORD
```

**Eufy の認証情報がコンテナに渡っていない**。

`compose.yaml` の `eufy-security-ws` セクションに、以下が必要。

* `USERNAME`
* `PASSWORD`
* `COUNTRY`
* `TRUSTED_DEVICE_NAME`
* `EVENT_DURATION_SECONDS`

設定変更後はコンテナを再作成する。

```bash
docker compose up -d
docker logs -f eufy-security-ws
```

---

## 8. パスワードに `$` が入っている場合の注意

Docker Compose では `$` が変数展開として解釈されることがある。
そのため、**`$` を含むパスワードはそのまま書かない**ほうがよい。

安全なのは次のように **シングルクォート**で囲み、必要に応じて `$$` を使う方法。

```yaml
PASSWORD: 'pa$$word$$with$$dollar'
```

---

## 9. HACS から Eufy Security 統合を入れる

HACS が有効になったら、

* **HACS → Integrations**
* `Eufy Security` を検索
* インストール
* Home Assistant を再起動

その後、

* **Settings → Devices & Services**
* **Add Integration**
* **Eufy Security**

を選んで追加する。

接続先は基本的に以下。

* Host: `127.0.0.1`
* Port: `3000`

---

## 10. Eufy アプリ側の設定

Home Assistant 側だけ設定しても、Eufy 側の設定が不十分だと検出イベントが来ない。

Eufy アプリで次を確認する。

* **Pet Detection** を ON
* **通知（push notification）** を ON
* 必要に応じて共有アカウント設定

---

## 11. 生成された entity を確認する

連携が成功すると、カメラに対して次のような entity が作られる。

```text
binary_sensor.living_room_pet_detected
image.living_room_event_image
```

名前は環境によって異なるので、実際の entity 名は Home Assistant UI で確認する。

* **Settings → Devices & Services**
* Eufy カメラを開く
* entity 一覧を見る

---

## 12. `automations.yaml` を作る

ペット検出時に画像を保存する automation は次のように書ける。

```yaml
alias: Eufy pet detection posts image to Slack
description: ""
triggers:
  - trigger: state
    entity_id:
      - image.living_room_event_image
conditions:
  - condition: state
    entity_id: binary_sensor.living_room_pet_detected
    state:
      - "on"
actions:
  - variables:
      ts: "{{ now().strftime('%Y%m%d-%H%M%S') }}"
      file: /media/eufy_snapshots/pet_{{ ts }}.jpg
  - action: image.snapshot
    metadata: {}
    target:
      entity_id: image.living_room_event_image
    data:
      filename: "{{ file }}"
```

`entity_id` は自分の環境の名前に置き換える。

---

## 13. `automations.yaml` を反映する

`automations.yaml` を直接編集した場合、**保存しただけでは自動反映されない**。

反映するには以下のどちらかを行う。

### 方法1: UI から再読み込み

* **Developer Tools**
* **Actions**
* `automation.reload` を実行

### 方法2: Home Assistant を再起動

```bash
docker compose restart homeassistant
```

---

## 14. automation が反映されたか確認する

確認方法は次の通り。

1. **Settings → Automations & Scenes → Automations** に alias が出る
2. automation を開いて **Run** できる
3. **Trace** が見られる
4. 保存先ディレクトリに画像ができる

保存先確認:

```bash
ls -lh ~/ha-stack/homeassistant/media/eufy_snapshots
```

---

## 15. 動作確認

最後に実際の流れを確認する。

1. ペットをカメラ前に通す
2. `binary_sensor.xxx_pet_detected` が `on` になるか確認
3. `image.xxx_event_image` が更新されるか確認
4. `/media/eufy_snapshots` に JPEG が保存されるか確認

これで目的の構成は完成。

---

# 質問回答まとめ

ここからは、セットアップ中に出やすい疑問点を Q&A 形式で整理する。

---

## Q1. Raspberry Pi 4 + Raspberry Pi OS + Docker で本当に組める？

**組める。**

ただし、前提は次の構成になる。

* Home Assistant OS ではなく **Home Assistant Container**
* Eufy 連携は **`eufy-security-ws` を別コンテナ**
* HACS から **`eufy_security`** を導入

つまり、Add-on 前提の Home Assistant OS とは少し構成が違う。

---

## Q2. `configuration.yaml` は追記？ 上書き？

**追記。上書きしない。**

`configuration.yaml` は既存設定があることが普通。
今回必要なのは `allowlist_external_dirs` だけなので、必要な箇所を追加する。

### `homeassistant:` がない場合

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

### `homeassistant:` がある場合

その中に追加する。

```yaml
homeassistant:
  name: Home
  time_zone: Asia/Tokyo
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

---

## Q3. HACS が Add Integration に出てこない

よくある原因は次の3つ。

1. **Home Assistant の再起動不足**
2. **ブラウザキャッシュの問題**
3. **HACS ファイル配置失敗**

対処は以下。

```bash
docker compose restart homeassistant
```

そのあとブラウザを hard refresh する。

* Windows/Linux: `Ctrl + F5`
* Mac: `Cmd + Shift + R`

さらに、HACS のファイルが存在するか確認する。

```bash
ls ~/ha-stack/homeassistant/config/custom_components/hacs
```

---

## Q4. `eufy-security-ws` のログに `Missing one of USERNAME or PASSWORD` と出る

これは **Eufy の認証情報がコンテナに渡っていない** 状態。

`compose.yaml` の `eufy-security-ws` に以下を追加する必要がある。

```yaml
environment:
  TZ: Asia/Tokyo
  USERNAME: 'your-eufy-mail@example.com'
  PASSWORD: 'your-password'
  COUNTRY: 'JP'
  TRUSTED_DEVICE_NAME: 'rpi4-ha'
  EVENT_DURATION_SECONDS: '10'
```

---

## Q5. PASSWORD に `$` が入っている場合は？

Docker Compose では `$` が特別扱いされることがある。
そのため、パスワードに `$` がある場合は注意が必要。

たとえばこう書く。

```yaml
PASSWORD: 'pa$$word$$with$$dollar'
```

最初はここでハマりやすい。

---

## Q6. `automations.yaml` が反映されているかどうかを確認したい

確認手順は次の通り。

1. `automation.reload` を実行する
2. **Settings → Automations & Scenes → Automations** を見る
3. 作成した alias が出ているか確認する
4. 手動で **Run** してみる
5. 保存先に画像ができるか確認する

`automations.yaml` に書いただけでは自動反映されない点は見落としやすい。

---

## Q7. ライブ映像も必要？

今回の目的が **ペット検出時に画像を保存すること** なら、まずは不要。
`Event Image` だけ使う構成のほうが軽く、Pi 4 でも扱いやすい。

ライブ映像や録画までやると、負荷や安定性の問題が増えるので、まずは静止画保存だけに絞るのがよい。

---

## Q8. 実運用で気をつけることは？

いくつかある。

* **Raspberry Pi OS は 64-bit**
* 保存先は microSD より **SSD 推奨**
* まずは **画像保存のみ**
* トラブル時は **Home Assistant ログ / eufy-security-ws ログ** を見る
* Eufy アプリ側の **通知設定** を忘れない

---

# まとめ

Raspberry Pi 4 と Raspberry Pi OS、Docker を使って、
**Eufy IndoorCam 2K Pan & Tilt のペット検出イベントを Home Assistant で受け取り、画像保存する**構成は十分実現できる。

重要なのは次の3点。

* Home Assistant は **Container 運用**
* Eufy 連携は **`eufy-security-ws` + `eufy_security`**
* 画像保存は **`Event Image` を `image.snapshot` で保存**

最初は HACS や認証情報、`automations.yaml` の再読み込みで少し詰まりやすいが、そこを超えれば構成自体は比較的素直に動く。

Pi 4 で始める用途としても、**「ペット検出時に静止画を保存する」** という目的なら十分現実的な構成だと思う。
