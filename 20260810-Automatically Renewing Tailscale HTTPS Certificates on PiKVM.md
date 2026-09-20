---
title: "PiKVM で Tailscale HTTPS 証明書を自動更新する"
description: ""
pubDatetime: 2026-08-10T07:08:34.109Z
updatedDate: 2026-08-31T14:55:39.031Z
---

`/etc/kvmd/nginx/ssl.conf` では、引き続き

```nginx
ssl_certificate /etc/kvmd/nginx/ssl/server.crt;
ssl_certificate_key /etc/kvmd/nginx/ssl/server.key;
```

を使用し、systemd タイマーで証明書の有効期限を確認して、必要な場合にのみこの 2 ファイルを更新する構成が適切です。

PiKVM の公式ドキュメントでも、Tailscale の証明書を `/etc/kvmd/nginx/ssl/server.{crt,key}` に配置し、グループを `kvmd-nginx` に設定したうえで `kvmd-nginx` を再起動する方法が説明されています。([Pikvm][1]) また、`tailscale cert` でファイルとして取得した証明書は自動更新されないため、ユーザー側で独自の更新処理を実装する必要があります。現在の CLI では `--min-validity` も公式に利用できます。([Tailscale][2])

### 構成

通常、構成は次のようになります。

```text
Tailscale
   │
   │ 100.x / MagicDNS
   ▼
PiKVM nginx :443
   │
   ├─ /etc/kvmd/nginx/ssl/server.crt
   └─ /etc/kvmd/nginx/ssl/server.key
```

`tailscale serve` は使用しません。

```bash
tailscale serve --https=443 off
```

証明書の更新処理は次のようになります。

```text
タイマーを 1 日 1 回実行
        │
        ▼
現在の server.crt を確認
        │
        ├─ FQDN が正しく、
        │  かつ有効期限が 30 日以上残っている
        │       → 何もしない
        │
        └─ 残り 30 日未満 / 証明書なし / ホスト名不一致
                │
                ▼
               rw
                │
                ▼
        tailscale cert
                │
                ▼
        証明書/鍵を検証
                │
                ▼
        nginx のファイルを置換
                │
                ▼
        nginx -t
                │
                ▼
        kvmd-nginx を再起動
                │
                ▼
               ro
```

Let’s Encrypt の証明書は 90 日間有効なので、有効期限の 30 日前から更新を試みれば十分な余裕があります。([Tailscale][3])

---

## 1. 更新スクリプト

`/usr/local/libexec/pikvm-tailscale-cert-renew` を作成します。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

export PATH=/usr/local/bin:/usr/bin

CERT="/etc/kvmd/nginx/ssl/server.crt"
KEY="/etc/kvmd/nginx/ssl/server.key"

# 30 日
MIN_VALIDITY_SECONDS=$((30 * 24 * 60 * 60))
TS_MIN_VALIDITY="720h"

TMP=""
MADE_RW=0


log() {
    echo "pikvm-tailscale-cert-renew: $*"
}


cleanup() {
    rc=$?

    trap - EXIT INT TERM

    rm -f "${CERT}.new" "${KEY}.new" 2>/dev/null || true

    if [[ -n "${TMP:-}" ]]; then
        rm -rf "$TMP"
    fi

    if (( MADE_RW )); then
        sync

        if ! ro; then
            log "ERROR: 読み取り専用ファイルシステムへの復元に失敗しました"
            rc=1
        fi
    fi

    exit "$rc"
}

trap cleanup EXIT INT TERM


#
# Tailscale の FQDN を取得
#
DOMAIN="$(
    tailscale status --json |
        jq -er '.Self.DNSName | rtrimstr(".") | select(length > 0)'
)"

log "Tailscale DNS 名: ${DOMAIN}"


#
# nginx が現在使用している証明書を確認
#
cert_is_current() {
    [[ -s "$CERT" ]] || return 1
    [[ -s "$KEY" ]] || return 1

    # ホスト名が一致しているか確認
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkhost "$DOMAIN" \
        >/dev/null 2>&1 || return 1

    # 有効期限が 30 日以上残っているか確認
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkend "$MIN_VALIDITY_SECONDS" \
        >/dev/null 2>&1 || return 1

    return 0
}


if cert_is_current; then
    log "証明書の有効期限は 30 日以上残っています。処理は不要です"
    exit 0
fi

log "証明書の更新が必要です"


#
# 一時ディレクトリには /tmp を使用。
# この時点ではルートファイルシステムはまだ RO。
#
TMP="$(mktemp -d /tmp/pikvm-tailscale-cert.XXXXXX)"


#
# 必要な場合にのみ PiKVM のルートファイルシステムを RW に切り替える。
#
ROOT_OPTS="$(findmnt -no OPTIONS /)"

case ",${ROOT_OPTS}," in
    *,rw,*)
        log "ルートファイルシステムはすでに読み書き可能です"
        ;;
    *)
        log "ルートファイルシステムを読み書き可能に切り替えます"
        rw
        MADE_RW=1
        ;;
esac


#
# Tailscale から証明書を取得。
#
# --min-validity=720h により、少なくとも 30 日間
# 有効な証明書を要求する。
#
log "${DOMAIN} の証明書を要求しています"

tailscale cert \
    --min-validity="$TS_MIN_VALIDITY" \
    --cert-file="$TMP/server.crt" \
    --key-file="$TMP/server.key" \
    "$DOMAIN"


#
# 取得した証明書を検証
#

# ホスト名
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkhost "$DOMAIN"

# 有効期限
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkend "$MIN_VALIDITY_SECONDS"

# 証明書と秘密鍵の公開鍵が同一であることを確認
if ! cmp -s \
    <(
        openssl x509 \
            -in "$TMP/server.crt" \
            -pubkey \
            -noout |
        openssl pkey \
            -pubin \
            -outform DER 2>/dev/null
    ) \
    <(
        openssl pkey \
            -in "$TMP/server.key" \
            -pubout \
            -outform DER 2>/dev/null
    )
then
    log "ERROR: 証明書と秘密鍵が一致しません"
    exit 1
fi


#
# 現在の証明書をバックアップ
#
if [[ -e "$CERT" ]]; then
    cp -a "$CERT" "$TMP/old.crt"
fi

if [[ -e "$KEY" ]]; then
    cp -a "$KEY" "$TMP/old.key"
fi


rollback() {
    log "証明書をロールバックしています"

    if [[ -e "$TMP/old.crt" ]]; then
        cp -a "$TMP/old.crt" "$CERT"
    else
        rm -f "$CERT"
    fi

    if [[ -e "$TMP/old.key" ]]; then
        cp -a "$TMP/old.key" "$KEY"
    else
        rm -f "$KEY"
    fi
}


#
# nginx 用のファイルを準備してからリネームする。
#
# nginx 自体はリロード/再起動されるまで古い証明書を保持し続けるため、
# 2 回のリネームの間に crt/key ファイルが一時的に一致しない状態になっても、
# 稼働中の nginx プロセスには影響しない。
#
install \
    -o root \
    -g kvmd-nginx \
    -m 0644 \
    "$TMP/server.crt" \
    "${CERT}.new"

install \
    -o root \
    -g kvmd-nginx \
    -m 0640 \
    "$TMP/server.key" \
    "${KEY}.new"

mv -f "${KEY}.new" "$KEY"
mv -f "${CERT}.new" "$CERT"


#
# PiKVM が生成した実際の nginx 設定を使用して検証
#
if ! nginx -t -c /run/kvmd/nginx.conf; then
    log "ERROR: nginx の設定テストに失敗しました"
    rollback
    exit 1
fi


#
# PiKVM 公式ドキュメントに従って再起動。
#
if ! systemctl restart kvmd-nginx; then
    log "ERROR: kvmd-nginx の再起動に失敗しました"

    rollback

    # 古い証明書を復元した後、復旧を試みる
    nginx -t -c /run/kvmd/nginx.conf || true
    systemctl restart kvmd-nginx || true

    exit 1
fi


log "証明書を正常にインストールしました"

openssl x509 \
    -in "$CERT" \
    -noout \
    -subject \
    -issuer \
    -dates

exit 0
```

この方法では、通常の日次処理で実行されるのは次の確認だけです。

```bash
openssl x509 -checkhost ...
openssl x509 -checkend ...
```

したがって、**ルートファイルシステムは RO のまま維持されます**。

`rw` に切り替わるのは、有効期限の残りが 30 日未満になった場合だけです。

さらに、`tailscale cert --min-validity=720h` を使用しているため、Tailscale に対しても「少なくとも 30 日間有効な証明書を返す」よう指定しています。このフラグは現在の Tailscale CLI 仕様に含まれています。([Tailscale][2])

---

## 2. systemd サービス

`/etc/systemd/system/pikvm-tailscale-cert-renew.service`

```ini
[Unit]
Description=PiKVM nginx 用 Tailscale TLS 証明書を更新
Wants=network-online.target
After=network-online.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
ExecStart=/usr/local/libexec/pikvm-tailscale-cert-renew
TimeoutStartSec=5min
```

`Requires=` に `kvmd-nginx.service` を追加する必要はありません。

理由は、証明書の破損によって `kvmd-nginx` が停止していたとしても、このユニットは独立して証明書を修復し、その後 `systemctl restart kvmd-nginx` を実行できるようにしておくべきだからです。

---

## 3. systemd タイマー

`/etc/systemd/system/pikvm-tailscale-cert-renew.timer`

```ini
[Unit]
Description=PiKVM 用 Tailscale TLS 証明書の定期チェック

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
RandomizedDelaySec=30min
AccuracySec=1min
Unit=pikvm-tailscale-cert-renew.service

[Install]
WantedBy=timers.target
```

ここでは意図的に `Persistent=true` を省略しています。

この構成では、90 日間有効な証明書について有効期限の 30 日前から更新を開始するため、デバイスの電源が切れている間に 1 回チェックできなくても問題ありません。起動後およそ 15～45 分以内に証明書がチェックされ、その後はおよそ 1 日に 1 回チェックされます。

---

## 4. インストール

以下のブロック全体を PiKVM の root シェルにコピー＆ペーストしてください。最初に `jq` をインストールし、上記と同じ内容の更新スクリプトと 2 つの systemd ユニットファイルを作成し、Tailscale Serve を無効化し、タイマーを **有効化すると同時に起動** し、更新サービスをその場で 1 回実行した後、最後にルートファイルシステムを RO に戻します。

```bash
(
    set -Eeuo pipefail
    trap 'ro >/dev/null 2>&1 || true' EXIT

    rw

    # Install jq before installing/enabling the renewal service.
    pacman -S --needed jq

    install -d -m 0755 /usr/local/libexec

    cat > /usr/local/libexec/pikvm-tailscale-cert-renew <<'PIKVM_RENEW_EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

export PATH=/usr/local/bin:/usr/bin

CERT="/etc/kvmd/nginx/ssl/server.crt"
KEY="/etc/kvmd/nginx/ssl/server.key"

# 30 days
MIN_VALIDITY_SECONDS=$((30 * 24 * 60 * 60))
TS_MIN_VALIDITY="720h"

TMP=""
MADE_RW=0


log() {
    echo "pikvm-tailscale-cert-renew: $*"
}


cleanup() {
    rc=$?

    trap - EXIT INT TERM

    rm -f "${CERT}.new" "${KEY}.new" 2>/dev/null || true

    if [[ -n "${TMP:-}" ]]; then
        rm -rf "$TMP"
    fi

    if (( MADE_RW )); then
        sync

        if ! ro; then
            log "ERROR: failed to restore read-only filesystem"
            rc=1
        fi
    fi

    exit "$rc"
}

trap cleanup EXIT INT TERM


#
# Get the Tailscale FQDN
#
DOMAIN="$(
    tailscale status --json |
        jq -er '.Self.DNSName | rtrimstr(".") | select(length > 0)'
)"

log "Tailscale DNS name: ${DOMAIN}"


#
# Check the certificate currently used by nginx
#
cert_is_current() {
    [[ -s "$CERT" ]] || return 1
    [[ -s "$KEY" ]] || return 1

    # Check whether the hostname matches
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkhost "$DOMAIN" \
        >/dev/null 2>&1 || return 1

    # Check whether at least 30 days remain
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkend "$MIN_VALIDITY_SECONDS" \
        >/dev/null 2>&1 || return 1

    return 0
}


if cert_is_current; then
    log "certificate is valid for more than 30 days; nothing to do"
    exit 0
fi

log "certificate renewal is required"


#
# Use /tmp for the temporary directory.
# The root filesystem is still RO at this point.
#
TMP="$(mktemp -d /tmp/pikvm-tailscale-cert.XXXXXX)"


#
# Switch the PiKVM root filesystem to RW only when necessary.
#
ROOT_OPTS="$(findmnt -no OPTIONS /)"

case ",${ROOT_OPTS}," in
    *,rw,*)
        log "root filesystem is already read-write"
        ;;
    *)
        log "switching root filesystem to read-write"
        rw
        MADE_RW=1
        ;;
esac


#
# Obtain the certificate from Tailscale.
#
# --min-validity=720h requests a certificate
# that is valid for at least 30 days.
#
log "requesting certificate for ${DOMAIN}"

tailscale cert \
    --min-validity="$TS_MIN_VALIDITY" \
    --cert-file="$TMP/server.crt" \
    --key-file="$TMP/server.key" \
    "$DOMAIN"


#
# Validate the obtained certificate
#

# hostname
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkhost "$DOMAIN"

# expiration
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkend "$MIN_VALIDITY_SECONDS"

# Verify that the certificate and private key have the same public key
if ! cmp -s \
    <(
        openssl x509 \
            -in "$TMP/server.crt" \
            -pubkey \
            -noout |
        openssl pkey \
            -pubin \
            -outform DER 2>/dev/null
    ) \
    <(
        openssl pkey \
            -in "$TMP/server.key" \
            -pubout \
            -outform DER 2>/dev/null
    )
then
    log "ERROR: certificate and private key do not match"
    exit 1
fi


#
# Back up the current certificate
#
if [[ -e "$CERT" ]]; then
    cp -a "$CERT" "$TMP/old.crt"
fi

if [[ -e "$KEY" ]]; then
    cp -a "$KEY" "$TMP/old.key"
fi


rollback() {
    log "rolling back certificate"

    if [[ -e "$TMP/old.crt" ]]; then
        cp -a "$TMP/old.crt" "$CERT"
    else
        rm -f "$CERT"
    fi

    if [[ -e "$TMP/old.key" ]]; then
        cp -a "$TMP/old.key" "$KEY"
    else
        rm -f "$KEY"
    fi
}


#
# Prepare the files for nginx, then rename them.
#
# nginx itself continues holding the old certificate until it is
# reloaded/restarted, so even if the crt/key files briefly do not match
# between the two renames, this does not affect the running nginx process.
#
install \
    -o root \
    -g kvmd-nginx \
    -m 0644 \
    "$TMP/server.crt" \
    "${CERT}.new"

install \
    -o root \
    -g kvmd-nginx \
    -m 0640 \
    "$TMP/server.key" \
    "${KEY}.new"

mv -f "${KEY}.new" "$KEY"
mv -f "${CERT}.new" "$CERT"


#
# Validate using the actual nginx configuration generated by PiKVM
#
if ! nginx -t -c /run/kvmd/nginx.conf; then
    log "ERROR: nginx configuration test failed"
    rollback
    exit 1
fi


#
# Restart according to the official PiKVM documentation.
#
if ! systemctl restart kvmd-nginx; then
    log "ERROR: kvmd-nginx restart failed"

    rollback

    # Attempt recovery after restoring the old certificate
    nginx -t -c /run/kvmd/nginx.conf || true
    systemctl restart kvmd-nginx || true

    exit 1
fi


log "certificate successfully installed"

openssl x509 \
    -in "$CERT" \
    -noout \
    -subject \
    -issuer \
    -dates

exit 0
PIKVM_RENEW_EOF
    chmod 0755 /usr/local/libexec/pikvm-tailscale-cert-renew

    cat > /etc/systemd/system/pikvm-tailscale-cert-renew.service <<'PIKVM_SERVICE_EOF'
[Unit]
Description=Renew Tailscale TLS certificate for PiKVM nginx
Wants=network-online.target
After=network-online.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
ExecStart=/usr/local/libexec/pikvm-tailscale-cert-renew
TimeoutStartSec=5min
PIKVM_SERVICE_EOF

    cat > /etc/systemd/system/pikvm-tailscale-cert-renew.timer <<'PIKVM_TIMER_EOF'
[Unit]
Description=Periodic Tailscale TLS certificate check for PiKVM

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
RandomizedDelaySec=30min
AccuracySec=1min
Unit=pikvm-tailscale-cert-renew.service

[Install]
WantedBy=timers.target
PIKVM_TIMER_EOF

    systemctl daemon-reload

    # PiKVM nginx owns HTTPS port 443; Tailscale Serve must be disabled.
    tailscale serve --https=443 off

    # Enable and immediately start the periodic timer.
    systemctl enable --now pikvm-tailscale-cert-renew.timer

    # Run one certificate check/renewal immediately as part of installation.
    systemctl start pikvm-tailscale-cert-renew.service

    ro
    trap - EXIT
)
```

ブロック完了後は PiKVM 自身の nginx がポート 443 で待ち受けます。`systemctl enable --now` を使用しているため、タイマーはすでに有効化・起動済みです。

PiKVM の公式ドキュメントでも、Tailscale の証明書を nginx に直接インストールする場合、`server.crt/server.key` を更新して `systemctl restart kvmd-nginx` を実行する方法が使用されています。([Pikvm][1])

---

## 5. 起動とステータス確認

```bash
systemctl start pikvm-tailscale-cert-renew.service
```

確認：

```bash
systemctl status pikvm-tailscale-cert-renew.service
```

```bash
journalctl \
    -u pikvm-tailscale-cert-renew.service \
    -n 100 \
    --no-pager
```

証明書：

```bash
openssl x509 \
    -in /etc/kvmd/nginx/ssl/server.crt \
    -noout \
    -subject \
    -issuer \
    -dates \
    -ext subjectAltName
```

成功し、次の内容が含まれていれば問題ありません。

```text
DNS:{hostname}.{tsnet}.ts.net
```

タイマーはインストールブロックによってすでに有効化・起動されています。

確認：

```bash
systemctl list-timers pikvm-tailscale-cert-renew.timer
```

### アクセス URL

この構成では、証明書名は次のようになります。

```text
{hostname}.{tsnet}.ts.net
```

したがって、ブラウザでは常に次を使用します。

```text
https://{hostname}.{tsnet}.ts.net/
```

`https://{hostname}/` または `https://100.x.x.x/` の場合、接続自体は nginx に到達する可能性がありますが、証明書名は一致しません。Tailscale も、HTTPS 証明書は完全修飾された `*.ts.net` 名を対象とするものであり、単純なホスト名に対する HTTPS 証明書ではないことを明示しています。([Tailscale][3])

つまり、この方法では **Serve を完全に排除し、ポート 443 を PiKVM 標準の nginx のみで処理し、通常時はファイルシステムを RO のまま維持し、実際に証明書の更新が必要になった場合にのみ RW に切り替えることができます**。また、`/run/kvmd/nginx.conf` の自動生成とも競合しません。

[1]: [https://pikvm.github.io/pikvm/tailscale/]\(https://pikvm.github.io/pikvm/tailscale/\) "Tailscale VPN - PiKVM Handbook"
[2]: [https://tailscale.com/docs/reference/tailscale-cli]\(https://tailscale.com/docs/reference/tailscale-cli\) "Tailscale CLI · Tailscale Docs"
[3]: [https://tailscale.com/docs/how-to/set-up-https-certificates]\(https://tailscale.com/docs/how-to/set-up-https-certificates\) "Enabling HTTPS · Tailscale Docs"
