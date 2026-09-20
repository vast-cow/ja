---
pubDatetime: 2026-04-24T10:52:11+09:00
title: "Home Assistant で Eufy の検出画像を Slack に自動投稿する"
description: "画像取得までは設定済みの状態から、Slack 通知を追加する手順 Eufy と Home Assistant の連携が済んでいて、すでに Event Image を取得できる 状態なら、次はその画像を Slack に投げるだけです。 この記事では、次の前提で進めます。 Eufy のカメラは Home…"
---

## 画像取得までは設定済みの状態から、Slack 通知を追加する手順

Eufy と Home Assistant の連携が済んでいて、すでに **Event Image を取得できる** 状態なら、次はその画像を Slack に投げるだけです。

この記事では、次の前提で進めます。

* Eufy のカメラは Home Assistant に連携済み
* `image.xxx_event_image` が使える
* 必要なら `binary_sensor.xxx_pet_detected` のような検出センサーも使える
* Eufy からの画像取得まではすでに設定済み 

やることはシンプルです。

1. Slack App を作る
2. Home Assistant に Slack 統合を追加する
3. 既存 automation に「Slack へ画像投稿」を足す
4. 添付時に詰まりやすい権限や YAML を調整する

---

# 手順

## 1. まず全体の流れを確認する

今回やる構成はこうです。

```text
Eufy Event Image 更新
  ↓
Home Assistant automation 起動
  ↓
画像を /media/eufy_snapshots に保存
  ↓
Slack にその画像を添付して投稿
```

すでに Eufy 側から Event Image を取れているなら、追加するのは **Slack 投稿部分**です。
既存構成では、`/media/eufy_snapshots` への保存まで進める流れが組めています。

---

## 2. Slack App を作る

Slack に Home Assistant から投稿するには、Slack App を1つ作ります。
これは「通知専用の bot」を作るイメージです。

### 手順

1. Slack API の管理画面で新しい App を作成
2. 投稿先ワークスペースを選択
3. **OAuth & Permissions** を開く
4. **Bot Token Scopes** を設定する

### 最低限入れておきたい scope

テキスト投稿だけでなく、**ファイル添付**までやるなら次を入れておくのが安全です。

* `chat:write`
* `chat:write.public`
* `chat:write.customize`
* `files:write`
* `channels:read`
* `groups:read`
* `mpim:read`
* `im:read`

今回の会話では、**テキストだけ送れるのにファイル付きだと失敗**するケースがあり、原因は `missing_scope` でした。`conversations.list` 実行時に、`channels:read, groups:read, mpim:read, im:read` が不足していると失敗します。

### 終わったらやること

scope を追加したら、**ワークスペースへ再インストール**します。
これをしないと新しい権限が反映されません。

---

## 3. Bot Token を控える

Slack App を再インストールすると、**Bot User OAuth Token** が発行されます。
`xoxb-...` で始まる token です。

これはあとで Home Assistant の Slack 統合に入れるので、控えておきます。

---

## 4. bot を投稿先チャンネルに参加させる

これも忘れやすいポイントです。

たとえば投稿先が `#pet-camera` なら、そのチャンネルで bot を招待します。

```text
/invite @あなたのbot名
```

bot がチャンネルに入っていないと、投稿や添付で失敗することがあります。

---

## 5. Home Assistant に Slack 統合を追加する

Home Assistant 側で Slack 統合を追加します。

### 画面の場所

* **Settings**
* **Devices & Services**
* **Add Integration**
* **Slack**

### 入れる内容

* **API Key**
  → Slack の Bot User OAuth Token

* **Default Channel**
  → 例: `#pet-camera`

* **Username / Icon**
  → 必要に応じて設定

ここで作られる `notify.xxx` が、あとで automation から呼ぶ通知サービスになります。

---

## 6. `notify.xxx` のサービス名を確認する

Slack 統合を追加すると、Home Assistant 内で通知サービス名が作られます。

例:

* `notify.myworkspace`
* `notify.slack_teamname`

この名前は環境依存なので、**Developer Tools → Actions** で `notify.` と打って確認しておくと確実です。

### ここでの注意

ワークスペース名に **漢字や日本語を使っている場合**、`notify.XXX` のサービス名が、期待したローマ字ではなく**中国語読みベースのアルファベット転写のような形**になることがあります。
見た目で推測せず、**必ず実際のサービス名を確認**したほうがよいです。
これは今回の会話の中で実際に引っかかったポイントでした。

---

## 7. 画像保存先が allowlist に入っているか確認する

Slack 添付でローカルファイルを送るには、保存先が Home Assistant に許可されている必要があります。

既存構成では、画像保存用に次の設定が入っています。

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

まだ入っていなければ `configuration.yaml` に追加し、Home Assistant を再起動します。

---

## 8. まずはテキスト投稿だけテストする

いきなり automation を触る前に、Slack 連携だけテストします。

**Developer Tools → Actions** から、作成された `notify.xxx` を使って実行します。

```yaml
message: "テスト投稿です"
title: "Home Assistant テスト"
target: "#pet-camera"
```

これでテキストだけ届けば、認証や基本設定は通っています。

---

## 9. 次に固定画像で添付テストする

次は画像ファイル付き投稿を試します。
いきなりテンプレート変数を使わず、**固定パスの画像**で試すのが切り分けしやすいです。

```yaml
message: "画像テスト"
title: "test image"
data:
  file:
    path: /media/eufy_snapshots/test.jpg
```

最初は `target` を省略して、Slack 統合の **Default Channel** に送る形でも構いません。

ここで通れば、添付の基本動作は OK です。

---

## 10. automation に Slack 投稿を追加する

ここから本番です。
すでに Event Image を保存する automation があるなら、その最後に Slack 通知を足します。

以下は例です。

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
  - delay:
      hours: 0
      minutes: 0
      seconds: 2
      milliseconds: 0
  - action: notify.YOUR_SLACK_SERVICE
    data:
      message: ""
      data:
        file:
          path: "{{ file }}"
```

### この automation の考え方

* Event Image が更新されたら起動
* ペット検出センサーが `on` のときだけ動作
* Event Image を JPEG として保存
* 少し待つ
* 保存したそのファイルを Slack に添付して投稿

Eufy 画像保存のベース部分は、すでに組まれている Event Image 保存構成をそのまま流用しています。

---

## 11. automation を反映する

`automations.yaml` を直接編集した場合、保存しただけでは反映されません。

### 方法

* **Developer Tools → Actions** で `automation.reload`
* もしくは Home Assistant 再起動

---

## 12. 実際に動作確認する

最後に実機確認です。

1. ペットや人物をカメラ前に映す
2. `binary_sensor.xxx_pet_detected` が `on` になるか確認
3. `image.xxx_event_image` が更新されるか確認
4. `/media/eufy_snapshots` に画像が増えるか確認
5. Slack に画像付きで投稿されるか確認

保存確認の例:

```bash
ls -lh ~/ha-stack/homeassistant/media/eufy_snapshots
```

---

# 引っ掛かりやすい点の補足

ここからは、実際に詰まりやすかったポイントをまとめます。

---

## 1. テキスト投稿はできるのに、ファイル添付だけ失敗する

この場合、かなり多いのは **Slack App の scope 不足**です。

実際に起きたエラーはこれでした。

* `Error getting channel ID`
* `missing_scope`
* `needed: channels:read,groups:read,mpim:read,im:read`

つまり、テキスト投稿に必要な `chat:write` はあるが、**チャンネル解決に必要な read 系 scope が足りない**状態です。

### 対処

以下を Bot Token Scopes に追加します。

* `channels:read`
* `groups:read`
* `mpim:read`
* `im:read`

さらに、ファイル添付のために `files:write` も入れておきます。

その後、**ワークスペースへ再インストール**します。
今回の会話では、`files:write` を追加し、WS へ再インストールしたことで解決しました。

---

## 2. `notify.XXX` のサービス名を見た目で決めない

Slack ワークスペース名に **日本語や漢字**を使っていると、Home Assistant 側の `notify.XXX` の生成名が想像とずれることがあります。

今回の会話では、**漢字のワークスペース名が中国語読みっぽいアルファベット転写になった**ため、意図した名前と違って見えるケースがありました。

### 対処

* 予想で書かない
* **Developer Tools → Actions** で `notify.` を打って確認する

これが確実です。

---

## 3. `Message malformed: extra keys not allowed @ data['0']`

これは Slack の問題ではなく、**YAML の形がおかしい**ときに出やすいエラーです。

特に多いのは、

* automation 全体を貼る場所に action 単体 YAML を貼った
* action 単体を貼る場所に automation 全体を貼った
* 先頭の `-` が不要な場所に配列形式で貼った

というケースです。

### 典型例

Automation UI の 1 件編集画面では、先頭に `-` を付けた**配列形式**をそのまま貼ると壊れやすいです。

#### NG になりやすい例

```yaml
- alias: ...
  trigger:
    ...
```

#### 1件編集ならこちら

```yaml
alias: ...
trigger:
  ...
condition:
  ...
action:
  ...
```

### Developer Tools で通知だけ試す場合

通知アクション単体を書くなら、automation 全体ではなく、最小限にします。

```yaml
action: notify.YOUR_SLACK_SERVICE
data:
  message: "画像テスト"
  title: "test image"
  data:
    file:
      path: /media/eufy_snapshots/test.jpg
```

貼る場所ごとに YAML の期待構造が違う、というのが本質です。

---

## 4. 画像保存はできるのに、添付だけ失敗する

この場合は次を順番に見ると切り分けしやすいです。

### 確認順

1. `files:write` があるか
2. `channels:read` などの read 系 scope があるか
3. bot が投稿先チャンネルに入っているか
4. `path` のファイルが本当に存在するか
5. `allowlist_external_dirs` に保存先が入っているか
6. `notify.xxx` のサービス名が正しいか

---

## 5. まず固定ファイルで試したほうがよい

テンプレート変数付きの automation でいきなり試すと、失敗時の原因が見えづらくなります。

最初は必ず、固定ファイルでこの形を試すのが安全です。

```yaml
message: "画像テスト"
title: "test image"
data:
  file:
    path: /media/eufy_snapshots/test.jpg
```

これが通ってから `{{ file }}` に置き換えると、問題が切り分けやすくなります。

---

## 6. `target` は最初は省略してもよい

チャンネル解決が絡むと切り分けが面倒になります。
そのため最初は `target` を省略し、Slack 統合の **Default Channel** に送る形で試すのがよいです。

これで通れば、あとから `target: "#pet-camera"` を足せます。

---

## 7. 画像保存直後は少し待ったほうが安定する

保存してすぐ添付すると、タイミングによってはファイルがまだ扱いきれていないことがあります。

そのため automation では、保存直後に少し待つのが無難です。

```yaml
- delay:
    seconds: 2
```

小さい差ですが、安定性は上がります。

---

# まとめ

Eufy からの画像取得まですでに設定できているなら、Slack への自動投稿は次の追加で実現できます。

1. Slack App を作る
2. scope を正しく付ける
3. Home Assistant に Slack 統合を追加する
4. 既存 automation に添付付き通知を足す

今回いちばん重要だったのは次の3点です。

* **ファイル添付時は `files:write` だけでなく read 系 scope も必要**
* **ワークスペース名が日本語だと `notify.xxx` 名が予想外になることがある**
* **`Message malformed: extra keys not allowed @ data['0']` は YAML の構造ミス**

Eufy 側の Event Image 保存までできていれば、残りは Slack 側の権限と Home Assistant の YAML を丁寧に合わせるだけです。
そこさえ押さえれば、**「検出画像を Slack に自動投稿する」** ところまでは比較的素直に組めます。
