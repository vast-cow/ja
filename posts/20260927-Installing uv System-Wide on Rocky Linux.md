---
pubDatetime: 2026-09-27T21:58:00+09:00
title: "Rocky Linux に uv をシステムワイドでインストールする"
description: "公式インストーラーの配置先を `/usr/local/bin` に指定し、全ユーザーから `uv` / `uvx` を実行できるようにする。あわせて、インストーラーによるユーザー個別の PATH 変更を抑止する。"
---

Rocky Linux で `uv` 自体を **system-wide** に入れて、全ユーザーから `uv` / `uvx` を使えるようにしたいなら、`/usr/local/bin` に入れる方法が素直です。

```bash
sudo dnf install -y curl

curl -LsSf https://astral.sh/uv/install.sh \
  | sudo env UV_INSTALL_DIR=/usr/local/bin UV_NO_MODIFY_PATH=1 sh
```

確認:

```bash
which uv
uv --version
which uvx
```

期待値は:

```text
/usr/local/bin/uv
/usr/local/bin/uvx
```

公式インストーラーは通常ユーザー配下へインストールしますが、`UV_INSTALL_DIR` で配置先を変更できます。`UV_NO_MODIFY_PATH=1` を付けると、root の `.bashrc` などへの PATH 書き換えも抑止できます。([Astral Docs](https://docs.astral.sh/uv/reference/installer/))

### 注意点

これは **`uv` コマンド本体を system-wide にする**設定です。`uv` が管理する Python やキャッシュ、`uv tool install` で入れるツールまで自動的に共有領域になるわけではありません。デフォルトでは、それらは実行ユーザーごとの `~/.local/share/uv` や `~/.cache/uv` などを使います。([Astral Docs](https://docs.astral.sh/uv/reference/storage/))

たとえば、

```bash
# user1
uv python install 3.13

# user2
uv python list
```

とした場合、基本的には `user1` と `user2` で管理領域が別です。

もし意図が **「uv本体だけでなく、uvがインストールするPythonやtoolも `/opt/uv` などに全ユーザー共通で置きたい」** なら、Rocky Linux 向けに `/etc/profile.d/uv.sh` まで含めた system-wide 構成を作るのがよいです。その場合は例えば `/opt/uv/{python,tools,cache}` のように分離して設計できます。
