---
pubDatetime: 2026-06-08T13:59:04+09:00
title: "DiXiM PlayがDIGAを見つけられるようにするSSDPプロキシ"
description: "概要 このSSDPプロキシは、DiXiM PlayからDIGAを見つけやすくするための補助ツールです。 DiXiM Playは、ネットワーク上の録画機器やメディアサーバーを探すときにSSDPという仕組みを使います。しかし、ネットワーク構成や機器の応答条件によっては、DiXiM PlayからDIGAが…"
---

## 概要

このSSDPプロキシは、DiXiM PlayからDIGAを見つけやすくするための補助ツールです。

DiXiM Playは、ネットワーク上の録画機器やメディアサーバーを探すときにSSDPという仕組みを使います。しかし、ネットワーク構成や機器の応答条件によっては、DiXiM PlayからDIGAがうまく見つからないことがあります。

このプロキシは、DiXiM Playからの探索要求を受け取り、DIGAなどの実機器に向けて探索を出し直します。そして、見つかった機器からの応答をDiXiM Playへ返します。

## 目的

このツールの目的は、DiXiM PlayとDIGAの間で機器探索がうまく届かない場合に、その橋渡しをすることです。

主に次のような場面で役立ちます。

* DiXiM PlayでDIGAが一覧に出てこない
* DIGAは同じネットワーク上にあるのに検出されない
* IPv6側から来た探索をIPv4のSSDP探索としてDIGA側へ流したい
* MediaServerとして検出される機器をより確実に探したい

## 使い方

このSSDPプロキシは、ネットワーク上で常時動かして使います。

通常は、DIGAと同じネットワークに接続されたRaspberry Piなどで動かします。プロキシを起動しておくと、DiXiM Playから送られる探索要求を受け取り、DIGAが応答できる形に変換して転送します。

基本的な流れは次のとおりです。

1. DiXiM Playがネットワーク上のメディアサーバーを探す
2. SSDPプロキシがその探索要求を受け取る
3. プロキシがIPv4のSSDP探索としてDIGA側へ送信する
4. DIGAから応答が返る
5. プロキシがその応答をDiXiM Playへ返す
6. DiXiM Play上でDIGAが見つかるようになる

## 設定する主な項目

このツールを使うときは、環境に合わせていくつかの設定を確認します。

### ネットワークインターフェース

使用するネットワークインターフェースを指定します。

たとえば、有線LANを使う場合は `eth0` を指定します。Raspberry Piで有線LAN接続している場合は、この設定で使えることが多いです。

### IPv4アドレス

プロキシを動かす機器のIPv4アドレスを設定します。

たとえば、Raspberry Piのアドレスが `192.168.40.2` の場合、そのアドレスを設定します。このアドレスは、DIGAと通信できるネットワーク上のものにする必要があります。

### 応答待ち時間

DIGAなどの機器からの応答を待つ時間を設定します。

短すぎると応答を取りこぼすことがあります。長すぎると不要な待ち時間が増えます。通常は数秒程度で十分です。

## 特徴

このSSDPプロキシには、DiXiM PlayからDIGAを見つけやすくするための工夫があります。

### IPv4とIPv6の探索を受けられる

DiXiM Playからの探索がIPv4でもIPv6でも受け取れるようになっています。

受け取った探索は、DIGA側に届きやすいIPv4のSSDP探索として送信されます。

### MediaServer:1からMediaServer:2も探せる

DiXiM Playが `MediaServer:1` を探している場合に、必要に応じて `MediaServer:2` も一緒に探せます。

これにより、機器の種類や応答の違いによって見つからないケースを減らせます。

### 重複した応答を抑制する

同じ機器から似た応答が複数返ってくる場合でも、短時間の重複応答は抑制されます。

これにより、DiXiM Play側に不要な応答が何度も返ることを防ぎます。

## 利用時の注意

このツールは、SSDPによる機器探索を補助するためのものです。DIGAの録画番組を直接再生したり、DLNA機能そのものを実装したりするものではありません。

また、ネットワークの設定によっては、マルチキャスト通信が制限されている場合があります。その場合は、ルーター、スイッチ、ファイアウォール、OS側の設定も確認する必要があります。

## まとめ

このSSDPプロキシは、DiXiM PlayがDIGAを見つけられない場合に、機器探索を補助するためのシンプルなヘルパーです。

DiXiM Playからの探索要求を受け取り、DIGA側へ適切に転送し、返ってきた応答をDiXiM Playへ返すことで、DIGAを検出しやすくします。

ネットワーク上でDIGAが存在しているのにDiXiM Playから見つからない場合、このプロキシを使うことで問題を回避できる可能性があります。

```python
#!/usr/bin/env python3
import asyncio
import fcntl
import logging
import socket
import struct
import time
from dataclasses import dataclass, field
from typing import Any


# ============================================================
# Configuration
# ============================================================

SSDP_PORT = 1900

MCAST_GRP_V4 = "239.255.255.250"
MCAST_GRP_V6 = "ff02::c"

IFACE_NAME = "wlp1s0"
# IFACE_NAME = "enp2s0"

# Obtain the IPv4 address from IFACE_NAME at runtime.
# Do not hardcode an IP address in the source code.

# Seconds to wait for responses from upstream
SESSION_TTL_SECONDS = 5.0

# When MediaServer:1 is requested, also search for MediaServer:2
EXPAND_MEDIASERVER_1_TO_2 = True

# Suppress duplicate forwarding of SSDP responses
RESPONSE_DEDUP_SECONDS = 3.0

# Cache duration when NOTIFY lacks CACHE-CONTROL: max-age
NOTIFY_CACHE_DEFAULT_SECONDS = 1800.0

# IPv4 multicast TTL
IPV4_MCAST_TTL = 4

# IPv6 multicast hop limit
IPV6_MCAST_HOPS = 10


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)


# ============================================================
# Data Structures
# ============================================================

@dataclass
class ClientSession:
    client_family: str              # "IPv4" or "IPv6"
    client_addr: Any                # IPv4: (ip, port), IPv6: (ip, port, flowinfo, scopeid)
    requested_st: str
    upstream_sts: set[str]
    expires_at: float
    created_at: float = field(default_factory=time.monotonic)


@dataclass
class CachedNotifyResponse:
    # Source IP of the NOTIFY. A NOTIFY with the same IP and NT overwrites the cached entry.
    source_ip: str
    st: str                         # NOTIFY NT. Output as ST in M-SEARCH responses.
    usn: str
    headers: dict[str, str]
    expires_at: float
    updated_at: float = field(default_factory=time.monotonic)


# ============================================================
# SSDP parse/build utility
# ============================================================

def parse_ssdp_packet(data: bytes) -> dict[str, str]:
    text = data.decode("utf-8", errors="replace")
    lines = text.replace("\r\n", "\n").split("\n")

    headers: dict[str, str] = {
        "_request_line": lines[0].strip() if lines else "",
    }

    for line in lines[1:]:
        line = line.strip()
        if not line:
            break

        if ":" not in line:
            continue

        k, v = line.split(":", 1)
        headers[k.strip().upper()] = v.strip()

    return headers


def get_method(headers: dict[str, str]) -> str:
    request_line = headers.get("_request_line", "")
    if not request_line:
        return ""

    return request_line.split(" ", 1)[0].upper()


def is_msearch_discover(headers: dict[str, str]) -> bool:
    method = get_method(headers)
    if method != "M-SEARCH":
        return False

    man = headers.get("MAN", "")
    if "ssdp:discover" not in man.lower():
        return False

    if not headers.get("ST", ""):
        return False

    return True


def is_notify(headers: dict[str, str]) -> bool:
    if get_method(headers) != "NOTIFY":
        return False

    # NOTIFY の search target 相当は NT。
    # これが無いものは M-SEARCH response に変換できない。
    return bool(headers.get("NT", ""))


def parse_cache_control_max_age(cache_control: str) -> float:
    """Extract max-age from CACHE-CONTROL in seconds."""
    for part in cache_control.split(","):
        key, sep, value = part.strip().partition("=")
        if sep and key.strip().lower() == "max-age":
            try:
                return max(0.0, float(value.strip().strip('"')))
            except ValueError:
                return NOTIFY_CACHE_DEFAULT_SECONDS

    return NOTIFY_CACHE_DEFAULT_SECONDS


def build_cache_control_max_age(seconds_remaining: float) -> str:
    return f"max-age={max(1, int(seconds_remaining))}"


def build_msearch_response_from_notify(
    headers: dict[str, str],
    cache_control_override: str | None = None,
) -> bytes | None:
    """
    Convert a NOTIFY into an HTTP/1.1 200 OK response for M-SEARCH.
    Always emit the NOTIFY NT as ST.
    """
    nt = normalize_st(headers.get("NT", ""))
    if not nt:
        return None

    if cache_control_override is not None:
        cache_control = cache_control_override
    else:
        cache_control = (
            headers.get("CACHE-CONTROL", "").strip()
            or f"max-age={int(NOTIFY_CACHE_DEFAULT_SECONDS)}"
        )

    response_lines: list[str] = [
        "HTTP/1.1 200 OK",
        f"CACHE-CONTROL: {cache_control}",
    ]

    # Preserve response-compatible headers from the NOTIFY.
    for key in ("DATE",):
        value = headers.get(key, "").strip()
        if value:
            response_lines.append(f"{key}: {value}")

    # Return the EXT header in the M-SEARCH response.
    response_lines.append("EXT:")

    for key in ("LOCATION", "SERVER"):
        value = headers.get(key, "").strip()
        if value:
            response_lines.append(f"{key}: {value}")

    # Important: explicitly use the NOTIFY NT as the ST in the M-SEARCH response.
    response_lines.append(f"ST: {nt}")

    for key in (
        "USN",
        "BOOTID.UPNP.ORG",
        "CONFIGID.UPNP.ORG",
        "SEARCHPORT.UPNP.ORG",
    ):
        value = headers.get(key, "").strip()
        if value:
            response_lines.append(f"{key}: {value}")

    response_lines.append("")
    response_lines.append("")
    return "\r\n".join(response_lines).encode("utf-8")


def normalize_st(st: str) -> str:
    return st.strip()


def expanded_upstream_sts(requested_st: str) -> set[str]:
    requested_st = normalize_st(requested_st)

    sts = {requested_st}

    if EXPAND_MEDIASERVER_1_TO_2:
        if requested_st == "urn:schemas-upnp-org:device:MediaServer:1":
            sts.add("urn:schemas-upnp-org:device:MediaServer:2")

    return sts


def response_matches_session(response_st: str, session: ClientSession) -> bool:
    response_st = normalize_st(response_st)

    if session.requested_st == "ssdp:all":
        return True

    return response_st in session.upstream_sts


def build_ipv4_msearch(original_headers: dict[str, str], upstream_st: str) -> bytes:
    """
    Rebuild an M-SEARCH received from Host A for IPv4 multicast.

    Even if received via IPv6, change HOST to 239.255.255.250:1900.
    Replace ST with upstream_st.
    Prefer the original MX and MAN values.
    """
    mx = original_headers.get("MX", "3").strip() or "3"
    man = original_headers.get("MAN", '"ssdp:discover"').strip() or '"ssdp:discover"'

    payload = (
        "M-SEARCH * HTTP/1.1\r\n"
        f"HOST: {MCAST_GRP_V4}:{SSDP_PORT}\r\n"
        f"MAN: {man}\r\n"
        f"MX: {mx}\r\n"
        f"ST: {upstream_st}\r\n"
        "\r\n"
    )

    return payload.encode("ascii")


def first_line(data: bytes) -> bytes:
    return data.splitlines()[0] if data.splitlines() else b""


def addr_ip(addr: Any) -> str:
    # IPv4: (ip, port), IPv6: (ip, port, flowinfo, scopeid)
    if isinstance(addr, tuple) and addr:
        return str(addr[0])
    return str(addr)


# ============================================================
# Interface utility
# ============================================================

def get_interface_ipv4(iface_name: str) -> str:
    """
    Retrieve the IPv4 address from a Linux network interface name.

    Example: iface_name="eth0" -> "192.168.40.2"
    """
    if not iface_name:
        raise ValueError("iface_name must not be empty")

    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        ifreq = struct.pack("256s", iface_name.encode("utf-8")[:15])
        result = fcntl.ioctl(sock.fileno(), 0x8915, ifreq)  # SIOCGIFADDR
        return socket.inet_ntoa(result[20:24])
    except OSError as e:
        raise RuntimeError(
            f"failed to get IPv4 address for interface {iface_name!r}"
        ) from e
    finally:
        sock.close()


# ============================================================
# SSDP Proxy core
# ============================================================

class SSDPProxy:
    def __init__(
        self,
        upstream_sock_v4: socket.socket,
        client_reply_sock_v4: socket.socket,
        client_reply_sock_v6: socket.socket,
    ):
        self.upstream_sock_v4 = upstream_sock_v4
        self.client_reply_sock_v4 = client_reply_sock_v4
        self.client_reply_sock_v6 = client_reply_sock_v6

        self.sessions: list[ClientSession] = []
        self.recent_responses: dict[tuple[Any, str, str], float] = {}

        # key = (source_ip, NT)
        # 要件: 同じ IP から同じ NT の NOTIFY が新たに来たら cache を更新する。
        self.notify_cache: dict[tuple[str, str], CachedNotifyResponse] = {}

    def cleanup(self) -> None:
        now = time.monotonic()

        self.sessions = [
            s for s in self.sessions
            if s.expires_at > now
        ]

        self.recent_responses = {
            k: expires_at
            for k, expires_at in self.recent_responses.items()
            if expires_at > now
        }

        old_count = len(self.notify_cache)
        self.notify_cache = {
            k: cached
            for k, cached in self.notify_cache.items()
            if cached.expires_at > now
        }
        expired_count = old_count - len(self.notify_cache)
        if expired_count:
            logging.info(
                "[notify cache] expired entries removed count=%d remaining=%d",
                expired_count,
                len(self.notify_cache),
            )

    def handle_client_msearch(
        self,
        family: str,
        data: bytes,
        addr: Any,
    ) -> None:
        self.cleanup()

        headers = parse_ssdp_packet(data)

        if is_notify(headers):
            self.cache_notify(
                data=data,
                addr=addr,
                source=f"{family} multicast",
            )
            return

        if not is_msearch_discover(headers):
            # Non M-SEARCH UDP packets are expected on SSDP multicast sockets.
            # Keep this at DEBUG so the default INFO log level stays quiet.
            logging.debug(
                "[%s client] ignored non-M-SEARCH/discover UDP from %r first_line=%r",
                family,
                addr,
                first_line(data),
            )
            return

        requested_st = normalize_st(headers["ST"])

        now = time.monotonic()

        upstream_sts = expanded_upstream_sts(requested_st)

        session = ClientSession(
            client_family=family,
            client_addr=addr,
            requested_st=requested_st,
            upstream_sts=upstream_sts,
            expires_at=now + SESSION_TTL_SECONDS,
        )
        self.sessions.append(session)

        logging.info(
            "[%s client] M-SEARCH from %r ST=%r -> upstream_sts=%r cache_entries=%d",
            family,
            addr,
            requested_st,
            sorted(upstream_sts),
            len(self.notify_cache),
        )

        # upstream に投げる前に、NOTIFY cache から即時応答する。
        self.reply_cached_notify_responses(session)

        for upstream_st in sorted(upstream_sts):
            payload = build_ipv4_msearch(headers, upstream_st)
            try:
                sent = self.upstream_sock_v4.sendto(
                    payload,
                    (MCAST_GRP_V4, SSDP_PORT),
                )
                logging.info(
                    "[upstream IPv4] sent M-SEARCH ST=%r bytes=%d for client=%r",
                    upstream_st,
                    sent,
                    addr,
                )
            except OSError as e:
                logging.exception(
                    "[upstream IPv4] failed to send M-SEARCH ST=%r for client=%r error=%s",
                    upstream_st,
                    addr,
                    e,
                )

    def cache_notify(
        self,
        data: bytes,
        addr: Any,
        source: str,
    ) -> None:
        """Cache a NOTIFY. Do not respond directly to the client here."""
        self.cleanup()

        headers = parse_ssdp_packet(data)
        if not is_notify(headers):
            return

        nt = normalize_st(headers.get("NT", ""))
        usn = headers.get("USN", "")
        nts = headers.get("NTS", "").strip().lower()
        source_ip = addr_ip(addr)
        cache_key = (source_ip, nt)

        if nts == "ssdp:byebye":
            removed = self.notify_cache.pop(cache_key, None)
            logging.info(
                "[notify cache] removed byebye source=%s ip=%r NT/ST=%r USN=%r existed=%s remaining=%d",
                source,
                source_ip,
                nt,
                usn,
                removed is not None,
                len(self.notify_cache),
            )
            return

        # Verify that conversion is possible first. ST will always be included.
        payload = build_msearch_response_from_notify(headers)
        if payload is None:
            logging.info(
                "[notify cache] ignored NOTIFY source=%s from=%r reason=unbuildable first_line=%r",
                source,
                addr,
                first_line(data),
            )
            return

        max_age = parse_cache_control_max_age(headers.get("CACHE-CONTROL", ""))
        if max_age <= 0:
            removed = self.notify_cache.pop(cache_key, None)
            logging.info(
                "[notify cache] ignored max-age<=0 source=%s ip=%r NT/ST=%r USN=%r removed=%s",
                source,
                source_ip,
                nt,
                usn,
                removed is not None,
            )
            return

        now = time.monotonic()
        updated = cache_key in self.notify_cache
        self.notify_cache[cache_key] = CachedNotifyResponse(
            source_ip=source_ip,
            st=nt,
            usn=usn,
            headers=dict(headers),
            expires_at=now + max_age,
            updated_at=now,
        )

        logging.info(
            "[notify cache] %s NOTIFY source=%s ip=%r NT/ST=%r USN=%r max_age=%.1fs entries=%d bytes=%d",
            "updated" if updated else "cached",
            source,
            source_ip,
            nt,
            usn,
            max_age,
            len(self.notify_cache),
            len(payload),
        )

    def reply_cached_notify_responses(self, session: ClientSession) -> None:
        """Immediately return cached NOTIFY entries as HTTP/1.1 200 OK responses to M-SEARCH."""
        self.cleanup()

        now = time.monotonic()
        matched = [
            cached for cached in self.notify_cache.values()
            if response_matches_session(cached.st, session)
        ]

        if not matched:
            logging.info(
                "[notify cache] miss client=%r ST=%r cache_entries=%d",
                session.client_addr,
                session.requested_st,
                len(self.notify_cache),
            )
            return

        logging.info(
            "[notify cache] hit client=%r ST=%r count=%d cache_entries=%d",
            session.client_addr,
            session.requested_st,
            len(matched),
            len(self.notify_cache),
        )

        for cached in matched:
            seconds_remaining = cached.expires_at - now
            if seconds_remaining <= 0:
                continue

            payload = build_msearch_response_from_notify(
                cached.headers,
                cache_control_override=build_cache_control_max_age(seconds_remaining),
            )
            if payload is None:
                logging.info(
                    "[notify cache] skipped unbuildable cached entry source_ip=%r ST=%r USN=%r",
                    cached.source_ip,
                    cached.st,
                    cached.usn,
                )
                continue

            try:
                if session.client_family == "IPv4":
                    sent = self.client_reply_sock_v4.sendto(
                        payload,
                        session.client_addr,
                    )
                elif session.client_family == "IPv6":
                    sent = self.client_reply_sock_v6.sendto(
                        payload,
                        session.client_addr,
                    )
                else:
                    logging.warning(
                        "[notify cache] unknown client family=%r client=%r",
                        session.client_family,
                        session.client_addr,
                    )
                    continue

                logging.info(
                    "[notify cache] sent cached response to [%s] %r source_ip=%r ST=%r USN=%r remaining=%.1fs bytes=%d",
                    session.client_family,
                    session.client_addr,
                    cached.source_ip,
                    cached.st,
                    cached.usn,
                    seconds_remaining,
                    sent,
                )

            except OSError as e:
                logging.exception(
                    "[notify cache] failed to send cached response to [%s] %r source_ip=%r ST=%r USN=%r error=%s",
                    session.client_family,
                    session.client_addr,
                    cached.source_ip,
                    cached.st,
                    cached.usn,
                    e,
                )

    def handle_upstream_response(
        self,
        data: bytes,
        addr: tuple[str, int],
    ) -> None:
        self.cleanup()

        headers = parse_ssdp_packet(data)
        request_line = headers.get("_request_line", "")

        # upstream socket 側にも NOTIFY が来た場合は cache へ入れる。
        if is_notify(headers):
            self.cache_notify(
                data=data,
                addr=addr,
                source="upstream IPv4",
            )
            return

        if not request_line.startswith("HTTP/"):
            logging.info(
                "[upstream IPv4] ignored non-response from %r first_line=%r",
                addr,
                first_line(data),
            )
            return

        response_st = normalize_st(headers.get("ST", ""))
        usn = headers.get("USN", "")

        if not response_st:
            logging.info(
                "[upstream IPv4] ignored response from %r reason=missing ST first_line=%r",
                addr,
                first_line(data),
            )
            return

        matched_sessions = [
            s for s in self.sessions
            if response_matches_session(response_st, s)
        ]

        if not matched_sessions:
            logging.info(
                "[upstream IPv4] response from %r ST=%r USN=%r has no active client session",
                addr,
                response_st,
                usn,
            )
            return

        logging.info(
            "[upstream IPv4] response from %r ST=%r USN=%r matched_sessions=%d",
            addr,
            response_st,
            usn,
            len(matched_sessions),
        )

        now = time.monotonic()

        for session in matched_sessions:
            dedup_key = (
                session.client_addr,
                response_st,
                usn,
            )

            if dedup_key in self.recent_responses:
                logging.info(
                    "[relay] duplicate response suppressed client=%r ST=%r USN=%r",
                    session.client_addr,
                    response_st,
                    usn,
                )
                continue

            self.recent_responses[dedup_key] = now + RESPONSE_DEDUP_SECONDS

            try:
                if session.client_family == "IPv4":
                    sent = self.client_reply_sock_v4.sendto(
                        data,
                        session.client_addr,
                    )
                elif session.client_family == "IPv6":
                    sent = self.client_reply_sock_v6.sendto(
                        data,
                        session.client_addr,
                    )
                else:
                    logging.warning(
                        "[relay] unknown client family=%r client=%r",
                        session.client_family,
                        session.client_addr,
                    )
                    continue

                logging.info(
                    "[relay] forwarded upstream response to [%s] %r ST=%r USN=%r bytes=%d",
                    session.client_family,
                    session.client_addr,
                    response_st,
                    usn,
                    sent,
                )

            except OSError as e:
                logging.exception(
                    "[relay] failed to forward to [%s] %r ST=%r USN=%r error=%s",
                    session.client_family,
                    session.client_addr,
                    response_st,
                    usn,
                    e,
                )


# ============================================================
# asyncio protocols
# ============================================================

class ClientMSearchProtocol(asyncio.DatagramProtocol):
    def __init__(self, proxy: SSDPProxy, family: str):
        self.proxy = proxy
        self.family = family

    def connection_made(self, transport: asyncio.BaseTransport) -> None:
        logging.info("[%s client] listener ready", self.family)

    def datagram_received(self, data: bytes, addr: Any) -> None:
        self.proxy.handle_client_msearch(self.family, data, addr)


class UpstreamResponseProtocol(asyncio.DatagramProtocol):
    def __init__(self, proxy: SSDPProxy):
        self.proxy = proxy

    def connection_made(self, transport: asyncio.BaseTransport) -> None:
        logging.info("[upstream IPv4] response listener ready")

    def datagram_received(self, data: bytes, addr: tuple[str, int]) -> None:
        self.proxy.handle_upstream_response(data, addr)


# ============================================================
# Socket creation
# ============================================================

def make_client_listen_socket_v4(interface_ipv4: str) -> socket.socket:
    """
    Receive IPv4 SSDP multicast M-SEARCH requests from Host A.
    """
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
    except OSError:
        pass

    sock.bind(("0.0.0.0", SSDP_PORT))

    mreq = socket.inet_aton(MCAST_GRP_V4) + socket.inet_aton(interface_ipv4)
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

    sock.setblocking(False)
    return sock


def make_client_listen_socket_v6(ifindex: int) -> socket.socket:
    """
    Receive IPv6 SSDP multicast M-SEARCH requests from Host A.
    """
    sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
    except OSError:
        pass

    try:
        sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)
    except OSError:
        pass

    sock.bind(("::", SSDP_PORT))

    group_bin = socket.inet_pton(socket.AF_INET6, MCAST_GRP_V6)
    mreq = group_bin + struct.pack("@I", ifindex)
    sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_JOIN_GROUP, mreq)

    sock.setblocking(False)
    return sock


def make_upstream_socket_v4(interface_ipv4: str, upstream_source_ipv4: str) -> socket.socket:
    """
    Socket used to send IPv4 M-SEARCH requests to real devices.
    Responses return to this socket's ephemeral port.
    """
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    # Send multicast using the interface IPv4 address.
    sock.setsockopt(
        socket.IPPROTO_IP,
        socket.IP_MULTICAST_IF,
        socket.inet_aton(interface_ipv4),
    )

    sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, IPV4_MCAST_TTL)

    # Even if the client listener receives multicast sent by this process,
    # the source port is an upstream ephemeral port, so it causes no downstream issues.
    # However, disable unnecessary loopback.
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_LOOP, 0)

    # Allocate a temporary port for upstream responses using port 0.
    sock.bind((upstream_source_ipv4, 0))

    sock.setblocking(False)
    return sock


def make_client_reply_socket_v4(interface_ipv4: str) -> socket.socket:
    """
    Socket for forwarding responses to IPv4 Host A.
    Bind to port 1900 so the source port is 1900.
    """
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
    except OSError:
        pass

    sock.bind((interface_ipv4, SSDP_PORT))
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_TTL, 1)

    return sock


def make_client_reply_socket_v6(ifindex: int) -> socket.socket:
    """
    Socket for forwarding responses to IPv6 Host A.
    source port を 1900 にしたいので [::]:1900 に bind する。
    """
    sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
    except OSError:
        pass

    try:
        sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)
    except OSError:
        pass

    sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_IF, ifindex)
    sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_HOPS, IPV6_MCAST_HOPS)

    sock.bind(("::", SSDP_PORT))

    return sock


# ============================================================
# main
# ============================================================

async def main() -> None:
    loop = asyncio.get_running_loop()

    ifindex = socket.if_nametoindex(IFACE_NAME)
    interface_ipv4 = get_interface_ipv4(IFACE_NAME)

    # upstream M-SEARCH の送信元 IPv4。通常は interface_ipv4 と同じ。
    upstream_source_ipv4 = interface_ipv4

    logging.info("interface %s index=%d ipv4=%s", IFACE_NAME, ifindex, interface_ipv4)

    client_listen_v4 = make_client_listen_socket_v4(interface_ipv4)
    client_listen_v6 = make_client_listen_socket_v6(ifindex)

    upstream_v4 = make_upstream_socket_v4(interface_ipv4, upstream_source_ipv4)

    client_reply_v4 = make_client_reply_socket_v4(interface_ipv4)
    client_reply_v6 = make_client_reply_socket_v6(ifindex)

    logging.info(
        "[upstream IPv4] source=%r multicast_target=%s:%d",
        upstream_v4.getsockname(),
        MCAST_GRP_V4,
        SSDP_PORT,
    )

    proxy = SSDPProxy(
        upstream_sock_v4=upstream_v4,
        client_reply_sock_v4=client_reply_v4,
        client_reply_sock_v6=client_reply_v6,
    )

    await loop.create_datagram_endpoint(
        lambda: ClientMSearchProtocol(proxy, "IPv4"),
        sock=client_listen_v4,
    )

    logging.info(
        "[IPv4 client] listening on 0.0.0.0:%d joined %s on %s",
        SSDP_PORT,
        MCAST_GRP_V4,
        interface_ipv4,
    )

    await loop.create_datagram_endpoint(
        lambda: ClientMSearchProtocol(proxy, "IPv6"),
        sock=client_listen_v6,
    )

    logging.info(
        "[IPv6 client] listening on [::]:%d joined [%s%%%s]",
        SSDP_PORT,
        MCAST_GRP_V6,
        IFACE_NAME,
    )

    await loop.create_datagram_endpoint(
        lambda: UpstreamResponseProtocol(proxy),
        sock=upstream_v4,
    )

    await asyncio.Event().wait()


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logging.info("stopped")
```
