---
pubDatetime: 2026-02-14T22:54:02+09:00
title: "GitHub ActionsでQEMU-KVMとSSHを用いてUbuntu Minimal Cloud Imageを実行する"
description: "本記事では、qemu-kvm-ubuntu-minimal-cloudimg-ssh というGitHub Actionsワークフローの内容を解説する。このワークフローは、QEMUとKVMを用いてUbuntu 24.04 Minimal Cloud Imageを仮想マシンとして起動し、SSH接続を行い…"
---

本記事では、**`qemu-kvm-ubuntu-minimal-cloudimg-ssh`** というGitHub Actionsワークフローの内容を解説する。このワークフローは、QEMUとKVMを用いてUbuntu 24.04 Minimal Cloud Imageを仮想マシンとして起動し、SSH接続を行い、起動確認を実施し、最後にクリーンアップまでを自動化する。

本ワークフローは `workflow_dispatch` により手動実行され、`ubuntu-latest` ランナー上で動作する。

---

## ワークフローの概要

ジョブ `boot-and-ssh` は、以下の手順を実行する。

1. 仮想化関連の依存パッケージをインストール
2. `/dev/kvm` へのアクセス確認
3. Ubuntu Minimalイメージのキャッシュ／取得
4. cloud-init 設定の生成
5. オーバーレイディスクの作成
6. KVM有効化でVM起動
7. SSHおよびcloud-init完了待機
8. SSH接続によるOS確認
9. VM停止およびログ保存

---

## 依存パッケージのインストール

以下のパッケージをインストールする。

* `qemu-system-x86`, `qemu-kvm`, `qemu-utils`
* `cloud-image-utils`, `genisoimage`
* `openssh-client`, `netcat-openbsd`, `curl`
* `util-linux`（`sg` コマンド用）

インストール後に以下を確認する。

* `sg` コマンドの存在
* QEMUのバージョン
* `/dev/kvm` の存在

これにより、ハードウェア仮想化が利用可能であることを検証する。

---

## KVMアクセスの確保

KVMアクセラレーションには `/dev/kvm` へのアクセスが必要である。

ワークフローでは以下を実施する。

* `/dev/kvm` の存在確認
* ユーザーを `kvm` グループへ追加
* `sg kvm -c 'command'` を使用して、CI環境内でグループ権限を適用

グループ追加は現在のシェルには即時反映されないため、`sg` を利用して実行時にグループ権限を有効化している。

---

## Ubuntu Minimalイメージの準備

使用するのは Ubuntu 24.04 (Noble) Minimal Cloud Image である。

```
https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
```

### キャッシュ戦略

`actions/cache@v4` を使用してイメージをキャッシュする。

* 再ダウンロードを防止
* CI実行時間を短縮
* OS名とイメージ名をキーに使用

キャッシュが存在しない場合のみ `curl` でダウンロードする。

---

## cloud-init の設定

cloud-init を用いてVMを自動初期化する。

### SSH鍵の生成

ED25519鍵を生成する。

* 秘密鍵: `id_ed25519`
* 公開鍵: `user-data` に埋め込み

### user-data

以下を定義する。

* `ubuntu` ユーザー作成
* パスワードなしsudo許可
* SSH鍵認証のみ有効
* rootログイン無効化
* `runcmd` により `/var/tmp/cloud-init-ready` を作成

このファイルは、初期化完了の判定に利用される。

### meta-data

以下を定義する。

* `instance-id`
* `local-hostname`

### Seed ISO作成

`cloud-localds` により `seed.iso` を生成する。

* `user-data`
* `meta-data`

このISOはVMにCD-ROMとして接続される。

---

## オーバーレイディスクの作成

ベースイメージを直接変更しないように、QCOW2オーバーレイを作成する。

* ファイル名: `ubuntu-minimal.qcow2`
* サイズ: 20GB
* backing file としてベースイメージを指定

これにより、CI環境での一時的利用が可能になる。

---

## QEMU + KVMでのVM起動

以下の構成で起動する。

* `-machine accel=kvm`
* `-cpu host`
* vCPU 2個
* メモリ 2GB
* virtioディスク／ネットワーク
* ポートフォワーディング:

```
hostfwd=tcp::2222-:22
```

つまり、

```
localhost:2222 → ゲストの22番ポート
```

`nohup` でバックグラウンド実行し、PIDを保存する。

起動直後にプロセスが終了した場合はログを表示して失敗させる。

---

## SSH待機処理

2段階で準備完了を確認する。

### 1. ポート確認

`nc` を用いて:

```
127.0.0.1:2222
```

への接続確認を実施。

### 2. cloud-init完了確認

SSHでログインし、以下を確認する。

```
/var/tmp/cloud-init-ready
```

これにより以下が保証される。

* OS起動完了
* cloud-init処理完了
* ユーザー作成完了

---

## OS確認

SSH経由で以下を実行する。

```
cat /etc/os-release
```

Ubuntu 24.04 Minimal が起動していることを確認する。

---

## クリーンアップ処理

`always()` 条件で実行される。

* PID読み取り
* `kill`
* 必要に応じて `kill -9`

これによりランナー上に不要なQEMUプロセスを残さない。

---

## ログのアップロード

`actions/upload-artifact@v4` を使用し、

```
vm/qemu.log
```

を保存する。

失敗時でもログは取得可能である。

---

## 設計上の特徴

| 項目              | 実装方法                  |
| --------------- | --------------------- |
| ハードウェアアクセラレーション | `/dev/kvm` + `sg kvm` |
| ベースイメージ保護       | QCOW2オーバーレイ           |
| 自動初期化           | cloud-init seed ISO   |
| 安全なSSH接続        | ED25519鍵・パスワード無効      |
| 起動完了判定          | `runcmd` によるマーカーファイル  |
| CI高速化           | GitHub Actionsキャッシュ   |
| 障害解析            | ログアップロード              |

---

## まとめ

本ワークフローは以下を実現する。

* GitHub Actions上でUbuntu Minimal VMを起動
* KVMアクセラレーションの活用
* cloud-initによる非対話的プロビジョニング
* SSHによる起動検証
* 確実なクリーンアップ

CI環境で仮想マシンの起動検証やOSテスト、インフラ関連の自動化を行うための再現性の高い構成例である。

```yaml
name: qemu-kvm-ubuntu-minimal-cloudimg-ssh

on:
  workflow_dispatch:

defaults:
  run:
    shell: /usr/bin/bash --noprofile --norc -e -o pipefail {0}

jobs:
  boot-and-ssh:
    runs-on: ubuntu-latest

    env:
      UBUNTU_MINIMAL_IMG_URL: https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
      UBUNTU_MINIMAL_IMG_NAME: ubuntu-24.04-minimal-cloudimg-amd64.img

    steps:
      - name: Install deps (qemu/kvm + cloud-init + sg)
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            util-linux \
            qemu-system-x86 qemu-utils qemu-kvm \
            cloud-image-utils genisoimage \
            openssh-client netcat-openbsd curl

          command -v sg
          qemu-system-x86_64 --version

      - name: Ensure KVM access via kvm group + sg
        run: |
          test -e /dev/kvm
          ls -l /dev/kvm
          id

          # Adding the user to the kvm group won't affect the current shell session immediately.
          sudo usermod -aG kvm "$USER" || true

          # Use sg to run a command with the kvm group in CI.
          sg kvm -c 'id && ls -l /dev/kvm'

      - name: Prepare workspace
        run: |
          mkdir -p vm/cache

      - name: Restore cached Ubuntu minimal image
        id: cache-ubuntu-img
        uses: actions/cache@v4
        with:
          path: vm/cache/${{ env.UBUNTU_MINIMAL_IMG_NAME }}
          key: ubuntu-minimal-${{ runner.os }}-${{ env.UBUNTU_MINIMAL_IMG_NAME }}

      - name: Download Ubuntu Minimal Cloud Image (only if cache miss)
        if: steps.cache-ubuntu-img.outputs.cache-hit != 'true'
        run: |
          curl -fsSL -o "vm/cache/${UBUNTU_MINIMAL_IMG_NAME}" "${UBUNTU_MINIMAL_IMG_URL}"
          ls -lh "vm/cache/${UBUNTU_MINIMAL_IMG_NAME}"

      - name: Prepare cloud-init seed + overlay disk
        run: |
          cd vm

          ssh-keygen -t ed25519 -N "" -f id_ed25519 <<<y >/dev/null 2>&1

          cat > user-data <<'EOF'
          #cloud-config
          users:
            - name: ubuntu
              sudo: ["ALL=(ALL) NOPASSWD:ALL"]
              groups: [sudo]
              shell: /bin/bash
              ssh_authorized_keys:
                - __SSH_PUBKEY__
          ssh_pwauth: false
          disable_root: true
          package_update: false
          packages: []
          runcmd:
            - [ sh, -c, "echo cloud-init-ready > /var/tmp/cloud-init-ready" ]
          EOF

          PUBKEY="$(cat id_ed25519.pub)"
          sed -i "s|__SSH_PUBKEY__|${PUBKEY}|g" user-data

          cat > meta-data <<'EOF'
          instance-id: gh-actions-qemu
          local-hostname: gh-actions-qemu
          EOF

          cloud-localds -v seed.iso user-data meta-data

          BASE_IMG="cache/${UBUNTU_MINIMAL_IMG_NAME}"
          test -f "${BASE_IMG}"
          qemu-img create -f qcow2 -F qcow2 -b "${BASE_IMG}" ubuntu-minimal.qcow2 20G

      - name: Boot VM with KVM (SSH forwarded to localhost:2222) [sg kvm + exec bash]
        run: |
          cd vm

          # Fail fast if port 2222 is already in use
          (ss -ltnp | grep ':2222' && echo "port 2222 already in use" && exit 1) || true

          : > qemu.log

          # Run QEMU under the kvm group using sg, and exec into a bash that runs the command sequence.
          sg kvm -c 'exec /usr/bin/bash -lc "
            set -euo pipefail

            nohup qemu-system-x86_64 \
              -machine accel=kvm \
              -cpu host \
              -smp 2 \
              -m 2048 \
              -nographic \
              -drive file=ubuntu-minimal.qcow2,format=qcow2,if=virtio \
              -drive file=seed.iso,media=cdrom \
              -netdev user,id=net0,hostfwd=tcp::2222-:22 \
              -device virtio-net-pci,netdev=net0 \
              >> qemu.log 2>&1 &

            echo \$! > qemu.pid
            test -s qemu.pid
          "'

          sleep 2
          PID="$(cat qemu.pid)"

          if ! ps -p "$PID" >/dev/null 2>&1; then
            echo "QEMU exited immediately. Dumping qemu.log:"
            echo "----------------------------------------"
            tail -n 200 qemu.log || true
            echo "----------------------------------------"
            exit 1
          fi

          ps -p "$PID" -o pid,cmd

      - name: Wait for SSH to become ready
        run: |
          cd vm

          for i in {1..120}; do
            if nc -z 127.0.0.1 2222; then
              echo "SSH port is open."
              break
            fi
            sleep 2
          done

          SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5"
          for i in {1..120}; do
            if ssh ${SSH_OPTS} -i id_ed25519 -p 2222 ubuntu@127.0.0.1 "test -f /var/tmp/cloud-init-ready"; then
              echo "cloud-init is ready."
              break
            fi
            sleep 2
          done

      - name: SSH and cat /etc/os-release
        run: |
          cd vm
          SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
          ssh ${SSH_OPTS} -i id_ed25519 -p 2222 ubuntu@127.0.0.1 "cat /etc/os-release"

      - name: Cleanup (stop QEMU)
        if: always()
        run: |
          if [ -f vm/qemu.pid ]; then
            PID="$(cat vm/qemu.pid || true)"
            if [ -n "${PID}" ] && ps -p "${PID}" >/dev/null 2>&1; then
              kill "${PID}" || true
              sleep 2
              ps -p "${PID}" >/dev/null 2>&1 && kill -9 "${PID}" || true
            fi
          fi

      - name: Upload qemu.log (always)
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: qemu-log
          path: vm/qemu.log
```
