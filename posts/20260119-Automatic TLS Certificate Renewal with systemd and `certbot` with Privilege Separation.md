---
pubDatetime: 2026-01-19T20:01:32+09:00
title: "systemd と Certbot による TLS 証明書自動更新（DynamicUser による権限分離）"
description: "セキュリティ設計（権限境界）、systemd による実行制御（最小・予測可能）、デプロイ時の安全策（無駄な再起動を避ける）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## はじめに

この構成は、Let's Encrypt の TLS 証明書取得・更新を Certbot で自動化し、systemd のサービスとタイマーで定期実行するものです。

セキュリティ上の主眼は「必要最小限の権限で動かす」ことにあります。

証明書取得や UPnP によるポート操作は systemd の `DynamicUser=yes` によって非特権ユーザーとして実行し、root 権限が必要な処理は Python パッケージの管理、証明書の配備、サービスの再起動に限定します。

固定の専用ユーザーは作成せず、`sudo -u` による明示的な権限切り替えも使用しません。

また、Python パッケージは毎回の実行前に更新を確認します。`uv` がインストールされている場合は `uv pip` を優先し、存在しない場合は `pip` にフォールバックします。

## セキュリティ設計（権限境界）

### DynamicUser による非特権実行

`DynamicUser=yes` を指定すると、systemd はサービスの起動時に一時的な UID/GID を割り当て、サービス終了時に解放します。

この仕組みにより、固定の専用ユーザーを管理する必要がなくなります。

非特権で実行する処理は以下です。

- `ip` と `jq` によるローカル IP アドレスの取得
- `upnpc` による UPnP ポートマッピングの追加・削除
- Certbot による ACME 認証と証明書の取得・更新
- 更新が成功したことを示すフラグファイルの作成

これらの処理には root 権限を与えません。

### root を使用する処理

root 権限が必要な処理は、systemd の実行コマンドに `+` プレフィックスを付けて分離します。

`ExecStartPre=+` は Certbot の Python 仮想環境を更新します。

`ExecStartPost=+` は取得した証明書を保護された配置先にコピーし、Nginx と StrongSwan を再起動します。

通常の `ExecStart=` は `DynamicUser=yes` による非特権ユーザーとして実行されます。

実行順序は次のとおりです。

```text
certbot-renew.timer
        |
        v
certbot-renew.service
        |
        +-- ExecStartPre=+update-certbot.sh
        |       root
        |       Python パッケージ更新
        |
        +-- ExecStart=certbot-renew.sh
        |       DynamicUser
        |       ローカル IP 取得
        |       UPnP ポート開放
        |       Certbot 実行
        |       UPnP ポート閉鎖
        |
        +-- ExecStartPost=+certbot-deploy.sh
                root
                更新フラグ確認
                証明書検証・配備
                Nginx / StrongSwan 再起動
```

`ExecStartPre=` または `ExecStart=` が失敗した場合、`ExecStartPost=` は実行されません。

なお、`+` プレフィックスは UID/GID の切り替えだけでなく、一部のサンドボックス制限も解除します。そのため、root で実行するスクリプトは root 所有とし、非特権ユーザーから変更できないようにします。

### systemd によるディレクトリ管理

永続データと一時データは systemd の以下のディレクティブで管理します。

- `StateDirectory=`：Certbot の設定、証明書、秘密鍵
- `LogsDirectory=`：Certbot のログ
- `RuntimeDirectory=`：一時作業領域、HTTP-01 チャレンジ、更新フラグ

`DynamicUser=yes` と `StateDirectory=` を組み合わせると、永続ディレクトリは systemd によって保護されます。

そのため、Nginx が外部から参照する HTTP-01 チャレンジ用の webroot は、保護された `StateDirectory` ではなく `RuntimeDirectory` に配置します。

### 更新が発生した場合だけデプロイする

Certbot の `--deploy-hook` でフラグファイルを作成します。

証明書が新規発行または更新された場合のみ、後続の root 処理で証明書をコピーし、サービスを再起動します。

証明書に変更がなければデプロイをスキップします。

## ファイル構成

プロジェクトは以下の構成とします。

```text
certbot-renewal/
├── certbot-renew.service
├── certbot-renew.timer
├── certbot.env.example
├── update-certbot.sh
├── certbot-renew.sh
├── certbot-deploy.sh
├── install.sh
└── uninstall.sh
```

インストール後の主要な配置先は以下です。

```text
/etc/
├── certbot-renewal/
│   └── certbot.env
└── systemd/system/
    ├── certbot-renew.service
    └── certbot-renew.timer

/usr/libexec/
└── certbot-renewal/
    ├── update-certbot.sh
    ├── certbot-renew.sh
    ├── certbot-deploy.sh
    └── venv/

/var/lib/
└── certbot-renewal/
    └── config/

/var/log/
└── certbot-renewal/

/run/
└── certbot-renewal/
    ├── work/
    ├── webroot/
    └── renewed
```

`/var/lib/certbot-renewal` と `/var/log/certbot-renewal` は、実際には systemd の private ディレクトリを介して管理されることがあります。

## certbot-renew.service

```ini
[Unit]
Description=Renew and deploy TLS certificates using Certbot
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot

EnvironmentFile=/etc/certbot-renewal/certbot.env

DynamicUser=yes

StateDirectory=certbot-renewal
StateDirectoryMode=0700

RuntimeDirectory=certbot-renewal
RuntimeDirectoryMode=0755

LogsDirectory=certbot-renewal
LogsDirectoryMode=0700

PrivateTmp=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes

# HTTP-01 challenge files must be readable by Nginx.
UMask=0022

ExecStartPre=+/usr/libexec/certbot-renewal/update-certbot.sh
ExecStart=/usr/libexec/certbot-renewal/certbot-renew.sh
ExecStartPost=+/usr/libexec/certbot-renewal/certbot-deploy.sh
```

`Type=oneshot` により、各フェーズは順番に実行されます。

`UMask=0022` は、非特権で実行する Certbot が作成する HTTP-01 チャレンジファイルを Nginx が読み取れるようにするための設定です。

一方、秘密鍵などの永続データは `StateDirectoryMode=0700` で保護します。Certbot 自体も秘密鍵に制限されたアクセス権限を設定します。

`NoNewPrivileges=yes` および `ProtectSystem=strict` によって通常の非特権処理を制限します。

## certbot-renew.timer

```ini
[Unit]
Description=Periodically renew TLS certificates using Certbot

[Timer]
Unit=certbot-renew.service

OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 12:00:00

RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

1 日 2 回 Certbot を実行します。

`RandomizedDelaySec=1h` により実行時刻を分散させ、`Persistent=true` により停止中に逃した実行を起動後に補います。

証明書の有効期限に十分な余裕がある場合、Certbot は更新せずに終了します。

## update-certbot.sh

このスクリプトは root で実行し、Certbot 専用の Python 仮想環境を作成または更新します。

実行のたびにパッケージ更新を確認するため、タイマー起動時にも毎回実行されます。

`uv` が利用できる場合は `uv pip` を優先します。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

VENV="$CERTBOT_VENV"

echo "[info] preparing Certbot virtual environment" >&2

if [[ ! -x "$VENV/bin/python" ]]; then
    echo "[info] creating virtual environment" >&2

    # The virtual environment must remain root-owned.
    umask 022
    python3 -m venv "$VENV"
fi

echo "[info] updating Python packages" >&2

if command -v uv >/dev/null 2>&1; then
    echo "[info] using uv pip" >&2

    uv pip install \
        --python "$VENV/bin/python" \
        --upgrade \
        certbot
else
    echo "[info] using pip" >&2

    if ! "$VENV/bin/python" -m pip --version \
        >/dev/null 2>&1; then
        "$VENV/bin/python" -m ensurepip --upgrade
    fi

    "$VENV/bin/python" -m pip install \
        --upgrade pip

    "$VENV/bin/python" -m pip install \
        --upgrade \
        --upgrade-strategy eager \
        certbot
fi

echo "[info] checking installed dependencies" >&2

if command -v uv >/dev/null 2>&1; then
    uv pip check --python "$VENV/bin/python"
else
    "$VENV/bin/python" -m pip check
fi

echo "[info] installed Certbot version:" >&2
"$VENV/bin/certbot" --version

echo "[info] Python package update completed" >&2
```

### uv と pip の使い分け

`uv pip install --python` を使うことで、対象の Python 仮想環境を明示的に指定できます。

`--upgrade` により Certbot と必要な依存パッケージの新しい互換バージョンを解決します。

通常の pip では、`--upgrade-strategy eager` も指定して依存パッケージの更新を積極的に行います。

いずれの場合も Certbot の依存関係に含まれない任意の Python パッケージまで更新するわけではありません。更新対象は Certbot とその依存パッケージです。

この構成では毎回パッケージ更新を試みるため、Python パッケージの取得元にアクセスできない場合や依存関係の解決に失敗した場合、証明書更新処理も失敗します。また、パッケージの最新リリースに不具合があると、証明書更新に影響する可能性があります。

本番環境では、このリスクを考慮して、エラー監視、更新前のバックアップ、必要に応じたバージョン固定を検討してください。

## certbot-renew.sh

Certbot と UPnP を非特権ユーザーで実行するスクリプトです。

root チェックや `sudo -u` による権限切り替えは必要ありません。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID == 0 )); then
    echo "[error] must run as non-root" >&2
    exit 1
fi

# Paths default to directories managed by systemd.
CONFIG_PATH="${CONFIG_PATH:-${STATE_DIRECTORY:?STATE_DIRECTORY is required}/config}"
WORK_DIR="${WORK_DIR:-${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/work}"
LOGS_DIR="${LOGS_DIR:-${LOGS_DIRECTORY:?LOGS_DIRECTORY is required}}"
WEB_ROOT="${WEB_ROOT:-${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/webroot}"
FLAGFILE="${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/renewed"

# Prepare writable runtime directories.
install -d -m 0700 "$WORK_DIR"
install -d -m 0755 "$WEB_ROOT"

echo "[info] detecting local IP address" >&2

local_ip="$(
    ip -j route get 1.1.1.1 |
        jq -r '.[0].prefsrc // empty'
)"

if [[ -z "$local_ip" ]]; then
    echo "[error] failed to detect local IP" >&2
    exit 1
fi

echo "[info] local IP: $local_ip" >&2

mapping_attempted=0

cleanup() {
    if (( mapping_attempted )); then
        echo "[info] closing UPnP port" >&2
        upnpc -d 80 tcp || true
    fi
}

trap cleanup EXIT
trap 'exit 143' TERM
trap 'exit 130' INT

echo "[info] opening UPnP port" >&2

# WAN:80 -> LAN:80, lease: 600 seconds
mapping_attempted=1
upnpc -a "$local_ip" 80 80 tcp 600

echo "[info] running Certbot" >&2

"$CERTBOT_VENV/bin/certbot" \
    certonly \
    --non-interactive \
    --quiet \
    --webroot \
    -w "$WEB_ROOT" \
    --cert-name "$CERT_NAME" \
    --key-type rsa \
    --rsa-key-size 4096 \
    -d "$DOMAINLIST" \
    --agree-tos \
    --no-eff-email \
    --email "$EMAIL" \
    --deploy-hook "/usr/bin/touch $FLAGFILE" \
    --config-dir "$CONFIG_PATH" \
    --work-dir "$WORK_DIR" \
    --logs-dir "$LOGS_DIR"

echo "[info] Certbot completed successfully" >&2
```

### UPnP ポートマッピングの扱い

HTTP-01 検証のため、一時的に WAN:80 から LAN:80 へのマッピングを追加します。

Certbot の終了時には `trap` によってマッピングを削除します。

ただし、SIGKILL、停電、プロセスの異常終了などでは cleanup が実行されない可能性があります。

このため、ルータ側が対応する場合はリース時間を 600 秒に設定し、マッピングが無期限に残らないようにしています。

また、`upnpc -d 80 tcp` は対象ポートのマッピングを削除する操作です。同じポートマッピングを他の用途で共用する構成には適しません。

### 更新フラグ

Certbot の `--deploy-hook` は証明書の発行または更新が成功した場合に実行されます。

このフックで以下のファイルを作成します。

```text
/run/certbot-renewal/renewed
```

フラグファイルのパスは `FLAGFILE="$RUNTIME_DIRECTORY/renewed"` として組み立てます。このファイルは `RuntimeDirectory` 内にあるため、サービス起動ごとに初期化されます。

後続のデプロイスクリプトは、このフラグの有無によって証明書を配置するかを判断します。

## certbot-deploy.sh

更新された証明書を Nginx と StrongSwan に配置する root 専用スクリプトです。

証明書に変更がなければ何もせず終了します。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

CONFIG_PATH="${CONFIG_PATH:-${STATE_DIRECTORY:?STATE_DIRECTORY is required}/config}"
FLAGFILE="${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/renewed"

if [[ ! -f "$FLAGFILE" ]]; then
    echo "[info] no certificate renewal; skip deploy" >&2
    exit 0
fi

echo "[info] deploying renewed certificates" >&2

CERTPATH="$CONFIG_PATH/live/$CERT_NAME"

CONFIG_REAL="$(realpath -e -- "$CONFIG_PATH")"
ARCHIVE_PREFIX="$CONFIG_REAL/archive/$CERT_NAME/"

check_cert_file() {
    local name="$1"
    local source

    source="$(realpath -e -- "$CERTPATH/$name")" \
        || return 1

    [[ "$source" == "$ARCHIVE_PREFIX"* ]] \
        || return 1

    [[ -f "$source" && -s "$source" ]] \
        || return 1

    printf '%s\n' "$source"
}

FULLCHAIN="$(check_cert_file fullchain.pem)" || exit 2
PRIVKEY="$(check_cert_file privkey.pem)" || exit 2
CHAIN="$(check_cert_file chain.pem)" || exit 2

echo "[info] preparing destination directories" >&2

install -d -m 0755 "$NGINX_CERTS"
install -d -m 0755 "$IPSEC_ETC/private"
install -d -m 0755 "$IPSEC_ETC/cacerts"
install -d -m 0755 "$IPSEC_ETC/certs"

echo "[info] installing Nginx certificates" >&2

install -m 0644 \
    "$FULLCHAIN" \
    "$NGINX_CERTS/fullchain.pem"

install -m 0600 \
    "$PRIVKEY" \
    "$NGINX_CERTS/privkey.pem"

echo "[info] installing StrongSwan certificates" >&2

install -m 0600 \
    "$PRIVKEY" \
    "$IPSEC_ETC/private/privkey.pem"

install -m 0644 \
    "$CHAIN" \
    "$IPSEC_ETC/cacerts/chain.pem"

install -m 0644 \
    "$FULLCHAIN" \
    "$IPSEC_ETC/certs/fullchain.pem"

echo "[info] validating Nginx configuration" >&2

nginx -t

echo "[info] restarting Nginx" >&2
systemctl restart nginx

echo "[info] restarting StrongSwan" >&2
systemctl restart strongswan-starter

echo "[info] certificate deployment completed" >&2
```

証明書の配備前には、必要なファイルが存在し、空でないことを確認します。

また、Certbot の `live/` 配下のシンボリックリンクが、想定する `archive/` 配下を指すことを確認します。

ただし、このファイルチェックは限定的な安全策です。非特権ユーザーが管理する証明書格納領域を root が読み取る以上、悪意のある同時変更やリンクの差し替えへの完全な防御にはなりません。

厳密な脅威モデルでは、証明書格納領域の扱い、ファイルディスクリプタを使った安全な読み取り、証明書と秘密鍵の整合性検証、サービス設定検証後の原子的な配備などを追加する必要があります。

また、この例はコピー途中の失敗に対する完全なロールバック機能を持ちません。実運用で可用性を重視する場合は、ステージングディレクトリと原子的な置換を検討してください。

## certbot.env.example

```ini
CERT_NAME=hoge.ddns.net
DOMAINLIST=hoge.ddns.net
EMAIL=hoge@mail.com

CERTBOT_VENV=/usr/libexec/certbot-renewal/venv

# Optional overrides (uncomment only when the defaults are unsuitable):
# CONFIG_PATH=/custom/certbot/config
# WORK_DIR=/custom/certbot/work
# LOGS_DIR=/custom/certbot/logs
# WEB_ROOT=/custom/certbot/webroot

NGINX_CERTS=/etc/nginx/pki/certs
IPSEC_ETC=/etc/ipsec.d
```

systemd が管理するディレクトリについては、インストール時に手動で所有者を設定する必要はありません。

`certbot.env` では、systemd が提供するディレクトリ環境変数から導出できるパスを通常は指定しません。各スクリプトで `${変数:-デフォルト値}` を使用し、**明示的な指定があればそれを優先**します。

| 設定項目 | 未指定時の値 | 参照元 |
| --- | --- | --- |
| `CONFIG_PATH` | `$STATE_DIRECTORY/config` | `StateDirectory=` |
| `WORK_DIR` | `$RUNTIME_DIRECTORY/work` | `RuntimeDirectory=` |
| `LOGS_DIR` | `$LOGS_DIRECTORY` | `LogsDirectory=` |
| `WEB_ROOT` | `$RUNTIME_DIRECTORY/webroot` | `RuntimeDirectory=` |
| `FLAGFILE`（内部変数） | `$RUNTIME_DIRECTORY/renewed` | `RuntimeDirectory=` |

`FLAGFILE` は設定ファイルで指定せず、常に `RUNTIME_DIRECTORY` から組み立てます。`CERTBOT_VENV`、`NGINX_CERTS`、`IPSEC_ETC` はこれらの systemd 管理ディレクトリから導出できないため、明示設定を維持します。

systemd が用意する環境変数には `RUNTIME_DIRECTORY`、`STATE_DIRECTORY`、`CACHE_DIRECTORY`、`LOGS_DIRECTORY`、`CONFIGURATION_DIRECTORY` があります。この構成では `CacheDirectory=` と `ConfigurationDirectory=` を使用していないため、対応する環境変数も参照しません。将来これらの管理対象に置くパスが増えた場合は、それぞれ `CACHE_DIRECTORY`、`CONFIGURATION_DIRECTORY` から同じ方法で導出します。複数のディレクトリを一つのディレクティブに指定した場合、環境変数はコロン区切りとなるため、本例では各管理ディレクトリを一つずつ指定する前提です。

手動でスクリプトを直接実行する場合、systemd が設定する環境変数は自動では与えられません。通常は unit 経由で実行してください。

`DOMAINLIST` はここでは単一ドメインを想定しています。

複数ドメインを指定する場合は、Certbot の `-d` オプションを複数渡す形式にスクリプトを拡張してください。

## install.sh

インストールスクリプトでは、コマンドの存在確認、ファイル配置、仮想環境の初期化、systemd タイマーの有効化を行います。

Python パッケージの更新処理は `update-certbot.sh` に集約するため、インストール時にも同じスクリプトを使用します。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

ETC_DIR=/etc/certbot-renewal
LIBEXEC_DIR=/usr/libexec/certbot-renewal
SYSTEMD_DIR=/etc/systemd/system

missing=()

for cmd in ip upnpc jq python3 systemctl nginx; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
        missing+=("$cmd")
    fi
done

if ! ip -j address >/dev/null 2>&1; then
    missing+=("iproute2 JSON support")
fi

if (( ${#missing[@]} > 0 )); then
    echo "[error] missing requirements:" >&2

    for item in "${missing[@]}"; do
        echo "  - $item" >&2
    done

    exit 1
fi

echo "[info] creating installation directories" >&2

install -d -m 0700 "$ETC_DIR"
install -d -m 0755 "$LIBEXEC_DIR"

echo "[info] installing configuration" >&2

if [[ ! -e "$ETC_DIR/certbot.env" ]]; then
    install -m 0600 \
        certbot.env.example \
        "$ETC_DIR/certbot.env"
else
    echo "[info] preserving existing configuration" >&2
fi

echo "[info] installing scripts" >&2

install -m 0755 \
    update-certbot.sh \
    certbot-renew.sh \
    certbot-deploy.sh \
    "$LIBEXEC_DIR/"

echo "[info] installing systemd units" >&2

install -m 0644 \
    certbot-renew.service \
    certbot-renew.timer \
    "$SYSTEMD_DIR/"

echo "[info] preparing Python virtual environment" >&2

# Import configuration for the initial package installation.
set -a
source "$ETC_DIR/certbot.env"
set +a

"$LIBEXEC_DIR/update-certbot.sh"

echo "[info] reloading systemd" >&2

systemctl daemon-reload

echo "[info] enabling renewal timer" >&2

systemctl enable --now certbot-renew.timer

echo "[info] installation completed" >&2
echo "[info] edit $ETC_DIR/certbot.env if needed" >&2
```

### インストール時の注意

環境設定ファイルが既に存在する場合、`install.sh` は上書きしません。

ただし、初回インストール時はサンプル値が配置されるため、実行前に設定内容を確認する必要があります。

また、Python の `venv` 機能が利用できる必要があります。Debian 系ディストリビューションでは、別途 `python3-venv` パッケージが必要になる場合があります。

`uv` は任意の依存関係であり、インストールされていなくても `pip` を利用できます。

## uninstall.sh

アンインストールでは、systemd タイマーとサービスを停止し、関連するスクリプト、仮想環境、unit ファイルを削除します。

誤って証明書や秘密鍵を失わないよう、通常のアンインストールでは Certbot の永続データと設定ファイルを保持します。

`--purge` を指定した場合のみ、設定・証明書・ログを含めて削除します。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

ETC_DIR=/etc/certbot-renewal
LIBEXEC_DIR=/usr/libexec/certbot-renewal
SYSTEMD_DIR=/etc/systemd/system

PURGE=false

case "${1:-}" in
    "")
        ;;
    --purge)
        PURGE=true
        ;;
    *)
        echo "Usage: $0 [--purge]" >&2
        exit 2
        ;;
esac

echo "[info] stopping renewal timer" >&2

systemctl disable --now certbot-renew.timer \
    2>/dev/null || true

echo "[info] stopping renewal service" >&2

systemctl stop certbot-renew.service \
    2>/dev/null || true

echo "[info] removing systemd units" >&2

rm -f -- \
    "$SYSTEMD_DIR/certbot-renew.service" \
    "$SYSTEMD_DIR/certbot-renew.timer"

systemctl daemon-reload
systemctl reset-failed certbot-renew.service \
    2>/dev/null || true

echo "[info] removing application scripts and venv" >&2

rm -rf -- "$LIBEXEC_DIR"

if [[ "$PURGE" == true ]]; then
    echo "[warn] purging Certbot configuration and certificates" >&2

    rm -rf -- "$ETC_DIR"

    # Remove both public paths and DynamicUser private backing data.
    rm -rf -- \
        /var/lib/certbot-renewal \
        /var/lib/private/certbot-renewal \
        /var/log/certbot-renewal \
        /var/log/private/certbot-renewal

    echo "[info] purge completed" >&2
else
    echo "[info] keeping configuration and certificates" >&2
    echo "[info] configuration: $ETC_DIR" >&2
    echo "[info] state: /var/lib/certbot-renewal" >&2
    echo "[info] logs: /var/log/certbot-renewal" >&2
fi

echo "[info] uninstall completed" >&2
```

### アンインストールの動作

通常のアンインストール：

```bash
sudo ./uninstall.sh
```

以下を削除します。

- systemd サービス・タイマーの unit ファイル
- インストールしたスクリプト
- Certbot 用 Python 仮想環境

証明書、秘密鍵、Certbot の更新履歴と設定ファイルは保持します。

完全削除：

```bash
sudo ./uninstall.sh --purge
```

通常削除の対象に加え、Certbot の設定ファイル、証明書、秘密鍵、ログを削除します。

**注意：** `--purge` は証明書の再取得に必要なアカウント情報や秘密鍵も失わせる可能性があります。実行前に必要なデータをバックアップしてください。

Nginx と StrongSwan に配備済みの証明書ファイルは削除しません。これらは他の設定から参照されている可能性があるため、別途確認して削除してください。

Nginx の設定変更も自動的には元に戻しません。

## Nginx の HTTP-01 設定

Certbot の webroot を `/run/certbot-renewal/webroot` に変更しているため、Nginx 側でも同じディレクトリを参照させます。

```nginx
server {
    listen 80;
    server_name hoge.ddns.net;

    location ^~ /.well-known/acme-challenge/ {
        root /run/certbot-renewal/webroot;
        default_type text/plain;
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

`RuntimeDirectory=certbot-renewal` により、サービス起動中は `/run/certbot-renewal` が存在します。

また、`RuntimeDirectoryMode=0755` と webroot の `0755` によって Nginx がチャレンジファイルを読み取れるようにしています。

なお、Nginx 自体に `ProtectSystem=` や `InaccessiblePaths=` などの独自のサンドボックス制限が設定されている場合は、Nginx から webroot にアクセスできることを確認してください。

HTTP-01 検証を行うには、外部から当該ドメインの TCP/80 番ポートに到達できる必要があります。

## インストール方法

まず必要なファイルを作成し、環境設定を編集します。

```bash
cp certbot.env.example certbot.env
```

`certbot.env` の内容を環境に合わせて編集し、インストールします。

```bash
sudo ./install.sh
```

インストールされた環境設定ファイルを確認します。

```bash
sudoedit /etc/certbot-renewal/certbot.env
```

初回実行は手動で行います。

```bash
sudo systemctl start certbot-renew.service
```

動作ログを確認します。

```bash
sudo journalctl \
    -u certbot-renew.service \
    -n 100 \
    --no-pager
```

タイマーの状態を確認します。

```bash
systemctl list-timers certbot-renew.timer
```

systemd unit の設定を確認する場合は以下を使用します。

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/certbot-renew.service
```

権限設定の評価には以下を使用できます。

```bash
systemd-analyze security certbot-renew.service
```

ただし、`systemd-analyze security` は静的な設定評価であり、実際の攻撃耐性を保証するものではありません。

## 既存環境からの移行

固定ユーザーを使用していた旧構成から移行する場合、Certbot の永続データを新しい `StateDirectory` に移行する必要があります。

事前に古い timer/service を停止し、証明書・秘密鍵・ACME アカウント情報・更新設定をバックアップします。

旧ディレクトリの内容を単純にコピーしただけでは、`DynamicUser=yes` で割り当てられたユーザーが正しくアクセスできない場合があります。

systemd の private ディレクトリ管理と所有権処理を考慮して移行し、動作確認後に旧構成を削除してください。

特に `live/` から `archive/` を参照するシンボリックリンクは、コピー方法によって壊れる可能性があるため注意が必要です。

## 運用上の注意点

### 毎回のパッケージ更新

この構成では、証明書更新を試行するたびに Certbot の Python パッケージ更新も確認します。

新しいパッケージの導入後に互換性の問題が発生すると、証明書更新が失敗する可能性があります。

継続運用では systemd の失敗ログを監視し、障害時に速やかに気付ける構成を推奨します。

また、パッケージの更新に必要な通信は、非特権で実行する ACME 通信とは別に root 権限で行われます。パッケージ取得元の信頼性と仮想環境の書き込み権限を厳格に管理する必要があります。

### 証明書の有効期限監視

systemd timer が正常に動作していても、DNS、ルータ、UPnP、Let's Encrypt 側の問題によって証明書更新が失敗することがあります。

timer の稼働確認だけでなく、実際の証明書の有効期限も監視することを推奨します。

### デプロイ先の管理

Nginx と StrongSwan の証明書ディレクトリは root が管理し、非特権ユーザーには書き込み権限を付与しません。

証明書の配備先が増えた場合も、非特権の Certbot 処理を変更せず、root のデプロイスクリプト側に処理を追加する構成を維持します。

### 外部サービスへの依存

証明書取得は Let's Encrypt の ACME サービス、Python パッケージ更新はパッケージインデックス、UPnP 操作はルータの実装に依存します。

これらの障害を完全に排除することはできないため、適切なログ記録と障害監視が必要です。

## まとめ

この構成では、systemd の `DynamicUser=yes` を利用して証明書更新処理を非特権ユーザーで実行し、root 権限が必要な処理だけを `ExecStartPre=+` と `ExecStartPost=+` に分離しました。

固定ユーザーを作成する必要がなくなり、所有権と実行時ディレクトリの管理を systemd に集約できます。

さらに、Certbot の Python 仮想環境を root 管理としたうえで、毎回の実行時に `uv pip` または `pip` で依存パッケージを更新します。

`install.sh` と `uninstall.sh` によって導入・撤去も管理し、通常のアンインストールでは証明書データを保持するようにしました。

この構成の要点は、証明書取得、Python パッケージ管理、本番環境への配備という異なる責務を明確に分離し、それぞれの処理に必要な権限だけを与えることです。
