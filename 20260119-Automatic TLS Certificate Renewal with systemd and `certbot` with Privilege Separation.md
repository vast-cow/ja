---
pubDatetime: 2026-01-19T20:01:32+09:00
title: "systemd と `certbot` による TLS 証明書自動更新（権限分離を重視）"
description: "この構成は Let’s Encrypt の証明書更新を certbot で自動化し、systemd のサービス／タイマーで定期実行します。セキュリティ上の主眼は「必要最小限の権限で動かす」ことにあり、root 権限はデプロイとサービス再起動など不可避な処理に限定し、外部要因の影響を受けやすい処理は非…"
---

この構成は Let’s Encrypt の証明書更新を `certbot` で自動化し、`systemd` のサービス／タイマーで定期実行します。セキュリティ上の主眼は「必要最小限の権限で動かす」ことにあり、root 権限はデプロイとサービス再起動など不可避な処理に限定し、外部要因の影響を受けやすい処理は非特権ユーザーへ確実に権限を落として実行します。

## セキュリティ設計（権限境界）

### root を使うのは“最後の一押し”だけ

スクリプトは `EUID==0` を要求しつつ、root が必要な範囲を明確に絞っています。root が必要なのは主に以下です。

* 保護された配置先（Nginx / StrongSwan の証明書ディレクトリ）への書き込み
* `systemctl` によるサービス再起動
* Nginx の設定検証（`nginx -t`）や最終デプロイ

逆に、証明書取得やネットワーク関連コマンドは root で実行しません。

### 外部に近い処理は非特権ユーザーで実行（権限ドロップ）

`EXECUTE_USER`（例: `acmebot`）を用意し、次の処理を明示的に `sudo -u "$EXECUTE_USER" -H` で実行します。

* UPnP によるポート開閉（`upnpc`）
* `certbot` の実行（仮想環境 `CERTBOT_VENV` 内）

これにより、ACME クライアントや依存ライブラリ、ルータ相手の UPnP 操作など「入力や外部環境に影響されやすい処理」を root 文脈から切り離し、万一の際の影響範囲を抑えます。

## systemd による実行制御（最小・予測可能）

### サービスユニット

`cert-deploy.service` は環境変数を `EnvironmentFile` から読み込み、スクリプトを起動します。`PrivateTmp=true` により、一時領域の分離が行われ、不要な干渉や情報漏えいのリスクを下げます。

### タイマーユニット

`cert-deploy.timer` は 1 日 2 回実行し、`RandomizedDelaySec=1h` により実行時間が分散されます。`Persistent=true` のため、停止中に逃した実行は起動後に追いつきます。

## デプロイ時の安全策（無駄な再起動を避ける）

### 更新があった場合だけデプロイ／再起動

`certbot` の `--deploy-hook` でフラグファイルを `touch` し、フラグが存在する場合のみデプロイと再起動を行います。更新がなければ終了するため、不要な再起動や構成変更リスクを避けられます。

### 証明書ファイルの健全性チェック

`fullchain.pem` / `privkey.pem` / `chain.pem` が「存在し」「空でない」ことを確認してからコピーします。破損・未生成・部分更新の状態を本番配置先へ伝播しないための基本防御です。

### Nginx は再起動前に設定検証

`nginx -t` を通してから再起動するため、設定エラーによる停止リスクを抑制できます。

## HTTP-01 検証の露出を最小化（UPnP の扱い）

HTTP-01 検証のため一時的に WAN:80 → LAN:80 を UPnP で開け、終了時（成功・失敗を問わず）`trap` で必ず閉じます。さらに UPnP 操作自体は非特権ユーザーで実行するため、外部境界に近い操作を root から遠ざけられます。

## 依存関係と実行環境の隔離

`install.sh` は要件コマンド（`iproute2` の JSON 出力、`upnpc`、`jq`、`python3`）を事前チェックし、不足があれば停止します。また `certbot` はシステム Python に入れず、専用の仮想環境（`CERTBOT_VENV`）に隔離して導入します。非特権ユーザーが書き込む必要のあるディレクトリ（config/work/log/webroot）はそのユーザーに所有させ、デプロイ先の保護ディレクトリは root が管理します。

## 実運用での追加の締め付け（簡潔）

* `EXECUTE_USER` はログイン不可・最小グループの専用アカウントにする。
* `/etc/acmebot-certbot/env` は root 所有・最小権限で保護する。
* 非特権ユーザーに Nginx/StrongSwan の証明書配置先への書き込み権限を与えない（コピーは root が行う）。
* さらに強固にするなら、systemd ユニットに追加のサンドボックス制約（読み取り専用化、書き込み許可パスの限定など）を検討する。

この構成の要点は、証明書取得とネットワーク操作を非特権で実行し、root は「検証済み成果物の配置」と「サービス制御」に限定することで、攻撃面と影響範囲を最小化している点です。


## `cert-deploy.service`

```ini
[Unit]
Description=cert & deploy
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/libexec/acmebot-certbot/cert-deploy.sh
EnvironmentFile=/etc/acmebot-certbot/env
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## `cert-deploy.sh`

```bash
#!/bin/bash

# Must run as root
if (( EUID != 0 )); then
  echo "This script must be run as root. Exiting."
  exit 1
fi

set -xe
set -o pipefail

FLAGFILE="/run/acmebot-certbot/deploy-flag.$(date "+%Y%m%d%H%M%S%N")"
CERTPATH="$CONFIG_PATH/live/$CERT_NAME"

local_ip=""

cleanup() {
  # Always close port even on failure
  echo "[info] upnp port close" 1>&2
  sudo -u"$EXECUTE_USER" -H upnpc -d 80 tcp || true
}
trap cleanup EXIT

echo "[info] getting local address (ip -j route get + jq)" 1>&2

# ip -j route get 1.1.1.1 returns a JSON array; take the first element's "prefsrc"
local_ip="$(ip -j route get 1.1.1.1 | jq -r '.[0].prefsrc // empty')"

if [ -z "$local_ip" ] || [ "$local_ip" = "null" ]; then
  echo "[error] failed to detect local_ip via ip -j route get" 1>&2
  exit 1
fi

echo "[info] local_ip=${local_ip}" 1>&2

# NOTE: This opens WAN:80 -> LAN:80. If you intend WAN:80 -> LAN:8080, change the first "80" to "8080".
echo "[info] upnp port open (WAN 80 -> LAN 80)" 1>&2
sudo -u"$EXECUTE_USER" -H upnpc -a "${local_ip}" 80 80 tcp 600

echo "[info] certbot (webroot)" 1>&2

# Run certbot and capture exit code without aborting the script immediately
set +e
sudo -u "$EXECUTE_USER" -H \
     "${CERTBOT_VENV}/bin/certbot" \
     certonly -q --webroot -w "$WEB_ROOT" \
     --cert-name "${CERT_NAME}" \
     --key-type rsa --rsa-key-size 4096 \
     -d "$DOMAINLIST" \
     --agree-tos --no-eff-email --email "$EMAIL" \
     --deploy-hook "touch '$FLAGFILE'" \
     --config-dir "$CONFIG_PATH" \
     --work-dir "$WORK_DIR" \
     --logs-dir "$LOGS_DIR"
CERTBOT_STATUS=$?
set -e

if [ $CERTBOT_STATUS -ne 0 ]; then
  echo "[warn] Certbot exited with code $CERTBOT_STATUS" 1>&2
  exit $CERTBOT_STATUS
fi

if [ ! -e "$FLAGFILE" ]; then
  echo "[info] cert not renewed. skip deploy/restart." 1>&2
  exit 0
fi

echo "[info] cert renewed. proceed deploy/restart." 1>&2
rm -f "$FLAGFILE"

# Ensure cert files exist and are non-empty
for f in fullchain.pem privkey.pem chain.pem; do
  if [ ! -s "${CERTPATH}/${f}" ]; then
    echo "[error] missing or empty ${CERTPATH}/${f}" 1>&2
    exit 2
  fi
done

echo "[info] put certs" 1>&2

# nginx
cp -t "${NGINX_CERTS}" "${CERTPATH}/fullchain.pem" "${CERTPATH}/privkey.pem"

# strongswan
cp "${CERTPATH}/privkey.pem" "${IPSEC_ETC}"/private/privkey.pem
cp "${CERTPATH}/chain.pem" "${IPSEC_ETC}"/cacerts/chain.pem
cp "${CERTPATH}/fullchain.pem" "${IPSEC_ETC}"/certs/fullchain.pem

echo "[info] nginx config test" 1>&2
nginx -t

echo "[info] restart nginx and strongswan" 1>&2
systemctl restart nginx
systemctl restart strongswan-starter
```

## `cert-deploy.timer`

```ini
[Unit]
Description=Run certbot_update_cert periodically

[Timer]
OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 12:00:00
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

## `env.sample`

```bash
EXECUTE_USER=acmebot

ETC_DIR=/etc/acmebot-certbot
LIBEXEC_DIR=/usr/libexec/acmebot-certbot
SYSTEMD_DIR=/etc/systemd/system
RUN_DIR=/run/acmebot-certbot

CERT_NAME=hoge.ddns.net
DOMAINLIST=hoge.ddns.net
EMAIL=hoge@mail.com

CERTBOT_VENV=/usr/libexec/acmebot-certbot/venv

CONFIG_PATH=/var/lib/acmebot-certbot/config
WORK_DIR=/run/acmebot-certbot/work
LOGS_DIR=/var/log/acmebot-certbot

WEB_ROOT=/var/lib/acmebot-certbot/webroot

NGINX_CERTS=/etc/nginx/pki/certs

IPSEC_ETC=/etc/ipsec.d
```

## `install.sh`

```bash
#!/bin/bash

# Must run as root
if (( EUID != 0 )); then
  echo "[error] This script must be run as root. Exiting."
  exit 1
fi

set -xe

missing=()

# Check if `ip -j address` runs successfully
if ! ip -j address >/dev/null 2>&1; then
    missing+=("ip -j address (iproute2)")
fi

# Check if upnpc exists
if ! command -v upnpc >/dev/null 2>&1; then
    missing+=("upnpc")
fi

# Check if jq exists
if ! command -v jq >/dev/null 2>&1; then
    missing+=("jq")
fi

# Check if python3 exists
if ! command -v python3 >/dev/null 2>&1; then
    missing+=("python3")
fi

# Final check
if [ "${#missing[@]}" -ne 0 ]; then
    echo "The following requirements are missing or not working:"
    for item in "${missing[@]}"; do
        echo " - $item"
    done
    exit 1
fi

. ./env

mkdir -p "$ETC_DIR"
cp -t "$ETC_DIR" ./env

mkdir -p "${RUN_DIR}"
chown "${EXECUTE_USER}" "${RUN_DIR}"

mkdir -p "${LIBEXEC_DIR}"
install --mode=0755 cert-deploy.sh "${LIBEXEC_DIR}"

if [[ ! -x "${CERTBOT_VENV}/bin/certbot" ]]; then
    if [[ ! -x "${CERTBOT_VENV}/bin/python" ]]; then
        if [[ ! -d "${CERTBOT_VENV}" ]]; then
            mkdir -p "${CERTBOT_VENV}"
        fi
        python3 -m venv "${CERTBOT_VENV}"
    fi
    "${CERTBOT_VENV}/bin/pip" install cffi==1.17.1 certbot
fi

mkdir -p "${CONFIG_PATH}"
chown "${EXECUTE_USER}" "${CONFIG_PATH}"

mkdir -p "${WORK_DIR}"
chown "${EXECUTE_USER}" "${WORK_DIR}"

mkdir -p "${LOGS_DIR}"
chown "${EXECUTE_USER}" "${LOGS_DIR}"

mkdir -p --mode=0755 "${WEB_ROOT}"
chown "${EXECUTE_USER}" "${WEB_ROOT}"

install --mode=0644 cert-deploy.service "${SYSTEMD_DIR}"
install --mode=0644 cert-deploy.timer "${SYSTEMD_DIR}"
systemctl daemon-reload

systemctl enable cert-deploy.timer
systemctl start cert-deploy.timer
```
