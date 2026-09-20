---
pubDatetime: 2026-02-17T15:28:45+09:00
title: "Rootless Docker コンテナ内で X11 アプリを使う（xauth で MIT-MAGIC-COOKIE を渡す）"
description: "Rootless Docker はホスト側の権限を絞れて便利ですが、GUI アプリ（X11）をコンテナから表示したいときに少しだけハマりどころがあります。 この記事では ホストの X サーバに対して、コンテナ側へ xauth の Cookie を渡し、rootless Docker コンテナ内から…"
---

Rootless Docker はホスト側の権限を絞れて便利ですが、GUI アプリ（X11）をコンテナから表示したいときに少しだけハマりどころがあります。
この記事では **ホストの X サーバに対して、コンテナ側へ `xauth` の Cookie を渡し**、rootless Docker コンテナ内から X11 アプリを起動する手順をまとめます。

## 前提

* ホストで X サーバが動いている（Linux の Xorg / Xwayland 等）
* コンテナからホストの X サーバへ TCP 接続できる（後述の `--network host` を使用）
* rootless Docker を利用している（ただし手順自体は通常 Docker でも概ね同じ発想）

---

## 全体像（何をしているか）

X11 のアクセス制御はざっくり次の2段です。

1. **どの DISPLAY に接続するか**（例: `10.0.2.2:10`）
2. **接続を許可する認証情報（MIT-MAGIC-COOKIE）を渡すか**

この記事の手順は、ホストが保持している `MIT-MAGIC-COOKIE-1` をホスト側で確認し、コンテナ側で `xauth add` して `.Xauthority` を作る、という流れです。

---

## 1. ホストで X11 の Cookie を確認する

まずホストで、今の DISPLAY に紐づく Cookie を確認します。

```bash
$ xauth list "$DISPLAY"
hostname/unix:10  MIT-MAGIC-COOKIE-1  0a1b2c...
```

ここで出てくる `0a1b2c...` が Cookie です。
あとでコンテナ側に `xauth add` するので、コピーしておきます。

> 補足
> `hostname/unix:10` の末尾 `:10` はディスプレイ番号です。環境によって `:0` だったり `:1` だったりします。

---

## 2. コンテナを起動する（rootless Docker）

次にコンテナを起動します。例として `wine` イメージを使っていますが、任意のイメージで OK です。

```bash
$ docker run -it --rm --name wine-x11 \
  --network host \
  -e DISPLAY=host.docker.internal:12.0 \
  -v ./root:/root \
  -v ./work:/work \
  wine bash
```

### オプションの意図

* `--network host`
  X サーバへ到達しやすくするためにホストネットワークを使用します（rootless 環境で特に手っ取り早い）。
* `-e DISPLAY=host.docker.internal:12.0`
  コンテナ内のデフォルト DISPLAY 設定。あとで実際に使う DISPLAY は手元環境に合わせて切り替えます。
* `-v ./root:/root`
  コンテナ内の `/root` を永続化しておくと `.Xauthority` を保持できて便利です（毎回 `xauth add` したくない場合）。
* `-v ./work:/work`
  作業ディレクトリ用（任意）。

> 注意
> `host.docker.internal` は環境によって効かない場合があります。その場合でも、後段の `DISPLAY=10.0.2.2:10` のように **ホスト側 IP を明示**すれば進められることが多いです。

---

## 3. コンテナ内で `xauth add` して Cookie を登録する

コンテナに入ったら、ホストで見た Cookie を使って `xauth add` します。

```bash
# xauth add 10.0.2.2:10 MIT-MAGIC-COOKIE-1 "${cookie}"
```

* `10.0.2.2:10` の部分は「コンテナから見たホスト」と「ディスプレイ番号」です。
* `"${cookie}"` にはホストで確認した `0a1b2c...` を入れます。

### よく見る warning について

実行時に以下が出ることがあります。

```
xauth:  file /root/.Xauthority does not exist
```

これは **単に `.Xauthority` がまだ無い**だけなので、**warning として無視して OK**です。
（`xauth add` によってファイルが新規作成されます。）

---

## 4. `xev` で疎通確認する

最後に X11 クライアントを起動して、画面が出るか確認します。ここでは `xev` を例にします。

```bash
# DISPLAY=10.0.2.2:10 xev
```

ウィンドウが開き、キー入力やマウスイベントが流れてくれば成功です 🎯

---

## うまくいかないときのチェックポイント

* **DISPLAY 番号が合っているか**

  * ホストの `xauth list "$DISPLAY"` の末尾（`:10` など）と合わせる
* **`xauth add` の宛先が合っているか**

  * `10.0.2.2` があなたの環境で「コンテナ → ホスト」になっているか
* **X サーバが TCP 接続を受け付けているか**

  * 環境によっては `unix domain socket` 前提で、TCP が閉じている場合があります
* **Wayland 環境の場合**

  * Xwayland 経由になっていることが多く、DISPLAY が `:0` ではなく `:1` などになりがちです

---

## まとめ

rootless Docker でも、ポイントは単純で、

* ホストの `MIT-MAGIC-COOKIE` を取り出す
* コンテナで `xauth add` して `.Xauthority` を作る
* 正しい `DISPLAY` を指定して X11 アプリを起動する

これだけで X11 アプリを動かせます。
