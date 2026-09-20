---
pubDatetime: 2026-01-19T19:59:19+09:00
title: "BUFFALO WSR-3200AX4S 管理画面の自動データ取得について"
description: "本記事では、BUFFALO WSR-3200AX4S の管理画面（Admin UI）に対して、Python を用いて自動的に情報を取得する仕組みの概要を、シンプルに解説します。設定ファイルの役割、処理の流れ、取得データの扱い方を中心に説明します。 全体概要 この仕組みは、ルーターの管理画面に自動ログ…"
---

本記事では、BUFFALO WSR-3200AX4S の管理画面（Admin UI）に対して、Python を用いて自動的に情報を取得する仕組みの概要を、シンプルに解説します。設定ファイルの役割、処理の流れ、取得データの扱い方を中心に説明します。

## 全体概要

この仕組みは、ルーターの管理画面に自動ログインし、内部情報を取得した後、安全にログアウトする一連の処理を非同期で実行します。
通信結果はすべて保存され、後から内容を確認・解析できるよう設計されています。

## 設定ファイル（config.toml）

設定は `config.toml` にまとめられています。

* `base_url`：ルーターの管理画面のURL
* `name`：管理者ユーザー名
* `password`：管理者パスワード

これにより、コード側では設定を直接書き換えずに、環境ごとの切り替えが可能です。

## ログイン処理の仕組み

### HTML からのトークン取得

ログイン画面（`login.html`）を取得し、HTML 内の特定の `<img>` タグに埋め込まれた Base64 データから `httoken` を抽出します。
このトークンは、認証リクエストに必須の要素です。

### 認証リクエスト

* パスワードは MD5 ハッシュ化されます
* `login.cgi` に対して POST リクエストを送信します
* 応答 HTML の `<title>` が期待値であることを確認し、正常ログインを検証します

## 情報取得処理

### info.html の取得

ログイン後に `info.html` を取得します。このページには `<title>` タグが存在しないことが前提条件となっています。
ここから再度 `httoken` を抽出します。

### cgi_info.js の解析

`cgi/cgi_info.js` を取得し、以下を確認します。

* Content-Type が `application/x-javascript` であること
* JavaScript 内の `addCfg(...)` 呼び出しを走査すること

特定条件（WAN DHCP DNS に関する設定）に一致するデータを見つけた場合、その内容をトークンとして分解・記録します。

## レスポンス保存の仕組み

すべての HTTP 通信結果は、以下の2種類のファイルとして保存されます。

* レスポンスボディ（HTML / JS / JSON など）
* メタ情報（ヘッダー、ステータス、送信データを含む JSON）

ファイル名には日時・HTTPメソッド・ステータスコードなどが含まれ、後から追跡しやすい構成です。

## ログアウト処理

処理完了後は `logout.html` と `logout.cgi` を利用してログアウトします。
ログアウト結果の `<title>` を検証し、正常終了を確認します。

## まとめ

このコードは、BUFFALO WSR-3200AX4S の管理画面を対象に、

* 自動ログイン
* 内部設定情報の取得と解析
* 通信内容の完全保存
* 安全なログアウト

を一貫して実行する仕組みです。
非同期処理と厳密な検証ルールにより、安定した自動データ取得を実現しています。

## `config.toml`

```toml
[client]
base_url = "http://192.168.1.1:80"
name = "admin"
password = "pass"
```

## `main.py`

```python
import asyncio
import base64
import hashlib
import json
import logging
import os
import re
from dataclasses import dataclass, replace
from datetime import datetime
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple
from urllib.parse import urlencode, urljoin, urlparse

import aiohttp
from lxml import html as lxml_html

# TOML loader: Python 3.11+ -> tomllib, else -> tomli
try:
    import tomllib  # type: ignore
except Exception:  # pragma: no cover
    tomllib = None  # type: ignore
    import tomli  # type: ignore


# ----------------------------
# Configuration
# ----------------------------
@dataclass(frozen=True)
class Config:
    base_url: str = "http://192.168.1.1:80"
    name: str = "admin"
    password: str = "pass"
    cookie: str = "lang=8; mobile=false; url="  # As specified (only "url=")

    @staticmethod
    def _load_toml(path: Path) -> Dict[str, Any]:
        if not path.exists():
            raise FileNotFoundError(f"TOML config file not found: {path}")

        data = path.read_bytes()
        if tomllib is not None:
            return tomllib.loads(data.decode("utf-8"))
        return tomli.loads(data.decode("utf-8"))

    @classmethod
    def from_toml(cls, path: Path) -> "Config":
        """
        Load base_url/name/password from TOML:
          [client]
          base_url = "..."
          name = "..."
          password = "..."
        """
        raw = cls._load_toml(path)
        client = raw.get("client", {})
        if not isinstance(client, dict):
            raise ValueError("Invalid TOML format: [client] must be a table")

        # Optional override with defaults
        base_url = client.get("base_url", cls.base_url)
        name = client.get("name", cls.name)
        password = client.get("password", cls.password)

        # Basic validation
        if not isinstance(base_url, str) or not base_url:
            raise ValueError("TOML: client.base_url must be a non-empty string")
        if not isinstance(name, str) or not name:
            raise ValueError("TOML: client.name must be a non-empty string")
        if not isinstance(password, str) or not password:
            raise ValueError("TOML: client.password must be a non-empty string")

        return replace(cls(), base_url=base_url, name=name, password=password)


# ----------------------------
# Logging
# ----------------------------
def setup_logging() -> None:
    logging.basicConfig(
        level=logging.DEBUG,
        format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
    )


logger = logging.getLogger("client")


# ----------------------------
# HTML helpers
# ----------------------------
def extract_title(html_text: str) -> Optional[str]:
    """
    Extract <title>...</title> text from HTML.
    Returns None if not found or parsing fails.
    """
    try:
        doc = lxml_html.fromstring(html_text)
        titles = doc.xpath("//title")
        if not titles:
            return None
        title_text = titles[0].text_content()
        return title_text.strip() if title_text is not None else ""
    except Exception:
        return None


def assert_title_equals(html_text: str, expected: str, *, context: str) -> None:
    """
    Validate that the HTML <title> equals the expected string.
    """
    log = logging.getLogger("validator")
    title = extract_title(html_text)
    if title != expected:
        log.error(
            "Validation failed (%s): unexpected <title>. expected=%r actual=%r",
            context,
            expected,
            title,
        )
        raise ValueError(
            f"Validation failed ({context}): unexpected <title> (expected={expected!r}, actual={title!r})"
        )

    log.info("Validation passed (%s): <title> matched %r", context, expected)


def assert_title_absent(html_text: str, *, context: str) -> None:
    """
    Validate that the HTML contains no <title> tag.
    """
    log = logging.getLogger("validator")
    title = extract_title(html_text)
    if title is not None:
        log.error("Validation failed (%s): <title> tag must be absent but was present: %r", context, title)
        raise ValueError(f"Validation failed ({context}): <title> tag must be absent but was present")

    log.info("Validation passed (%s): <title> tag is absent", context)


def normalize_content_type(content_type: str) -> str:
    """
    Normalize Content-Type by removing parameters and lowercasing.
    Example: 'application/x-javascript; charset=UTF-8' -> 'application/x-javascript'
    """
    return (content_type or "").split(";", 1)[0].strip().lower()


def assert_content_type_equals(content_type: str, expected: str, *, context: str) -> None:
    """
    Validate that Content-Type matches expected (ignoring parameters).
    """
    log = logging.getLogger("validator")
    actual_norm = normalize_content_type(content_type)
    expected_norm = normalize_content_type(expected)
    if actual_norm != expected_norm:
        log.error(
            "Validation failed (%s): unexpected Content-Type. expected=%r actual=%r",
            context,
            expected_norm,
            actual_norm,
        )
        raise ValueError(
            f"Validation failed ({context}): unexpected Content-Type (expected={expected_norm!r}, actual={actual_norm!r})"
        )

    log.info("Validation passed (%s): Content-Type matched %r", context, expected_norm)


# ----------------------------
# httoken extraction
# ----------------------------
def get_httoken(html_text: str) -> str:
    """
    Input: HTML text
    Find: <img title="spacer" src="data:image/gif;base64,..." border="0">
    Return: base64.b64decode(datauri[78:]).decode()
    """
    log = logging.getLogger("get_httoken")

    doc = lxml_html.fromstring(html_text)
    imgs = doc.xpath("//img[@title='spacer']")
    log.debug("Found %d img elements with title='spacer'", len(imgs))
    if not imgs:
        raise ValueError("No <img> tag with title='spacer' was found")

    target = None
    for img in imgs:
        src = img.get("src") or ""
        if src.startswith("data:image/gif;base64,"):
            target = img
            break
    if target is None:
        target = imgs[0]

    datauri = target.get("src") or ""
    log.debug("Selected img src length=%d, head=%r", len(datauri), datauri[:120])

    # As specified: try datauri[78:] first
    try:
        token = base64.b64decode(datauri[78:]).decode()
        log.debug("Decoded token length=%d (using datauri[78:])", len(token))
        return token
    except Exception as e:
        log.exception("Decode failed with datauri[78:]: %s", e)

    # Debug fallback (out of spec, but helpful for diagnosis)
    prefix = "data:image/gif;base64,"
    if datauri.startswith(prefix):
        token = base64.b64decode(datauri[len(prefix) :]).decode()
        log.debug("Decoded token length=%d (fallback: strip data URI prefix)", len(token))
        return token

    raise ValueError("Failed to decode httoken")


def md5_hex(s: str) -> str:
    return hashlib.md5(s.encode()).hexdigest()


# ----------------------------
# Utility: safe filename
# ----------------------------
_SAFE_RE = re.compile(r"[^A-Za-z0-9._-]+")


def safe_filename(s: str, max_len: int = 120) -> str:
    s = _SAFE_RE.sub("_", s).strip("_")
    return (s[:max_len] if len(s) > max_len else s) or "root"


# ----------------------------
# Reproduce request body (form submit)
# ----------------------------
def build_request_body_bytes(data: Optional[Dict[str, Any]]) -> bytes:
    """
    Reproduce the encoded body that aiohttp would generate (for saving/debugging).
    Current behavior assumes form(dict) and produces application/x-www-form-urlencoded.
    """
    if not data:
        return b""
    return urlencode(data, doseq=True).encode("utf-8")


# ----------------------------
# Save response (body + meta JSON)
# ----------------------------
class ResponseSaver:
    def __init__(self, out_dir: Path) -> None:
        self.out_dir = out_dir
        self.out_dir.mkdir(parents=True, exist_ok=True)
        self.seq = 0
        self.log = logging.getLogger("saver")

    def _make_stem(self, method: str, url: str, status: int) -> str:
        self.seq += 1
        ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
        p = urlparse(url)
        path_part = safe_filename(p.path.lstrip("/") or "root")
        query_part = safe_filename(p.query) if p.query else ""
        if query_part:
            path_part = f"{path_part}__q_{query_part}"
        return f"{ts}_{self.seq:04d}_{method.upper()}_{status}_{path_part}"

    def _guess_ext(self, content_type: str) -> str:
        ct = (content_type or "").lower()
        if "text/html" in ct:
            return ".html"
        if "application/json" in ct or ct.endswith("+json"):
            return ".json"
        if ct.startswith("text/"):
            return ".txt"
        return ".bin"

    def save(
        self,
        *,
        method: str,
        url: str,
        status: int,
        reason: str,
        content_type: str,
        # request
        request_headers: Any,
        request_body: bytes,
        request_data: Optional[Dict[str, Any]],
        # response
        response_headers: Any,
        response_raw_headers: Any,
        response_body: bytes,
    ) -> Dict[str, Path]:
        stem = self._make_stem(method, url, status)

        body_ext = self._guess_ext(content_type)
        body_path = self.out_dir / f"{stem}{body_ext}"
        meta_path = self.out_dir / f"{stem}.headers.json"

        body_path.write_bytes(response_body)

        req_headers_map = dict(request_headers)
        req_headers_list = [[k, v] for k, v in req_headers_map.items()]

        req_body_text: Optional[str] = None
        if request_body:
            try:
                req_body_text = request_body.decode("utf-8")
            except Exception:
                req_body_text = None

        req_body_b64: Optional[str] = None
        if request_body:
            req_body_b64 = base64.b64encode(request_body).decode("ascii")

        resp_raw_list = []
        try:
            for k, v in response_raw_headers:
                resp_raw_list.append([k.decode(errors="replace"), v.decode(errors="replace")])
        except Exception:
            resp_raw_list = []

        set_cookie_headers: List[str] = []
        try:
            set_cookie_headers = response_headers.getall("Set-Cookie", [])
        except Exception:
            pass

        meta = {
            "saved_at": datetime.now().isoformat(),
            "method": method.upper(),
            "url": url,
            "request": {
                "headers_map": req_headers_map,
                "headers_list": req_headers_list,
                "body_bytes": len(request_body),
                "body_text_utf8": req_body_text,
                "body_base64": req_body_b64,
                "form_data": request_data or None,
            },
            "response": {
                "status": status,
                "reason": reason,
                "content_type": content_type,
                "headers_map": dict(response_headers),
                "headers_raw": resp_raw_list,
                "set_cookie": set_cookie_headers,
                "body_file": str(body_path.name),
                "body_bytes": len(response_body),
            },
        }

        meta_path.write_text(json.dumps(meta, ensure_ascii=False, indent=2), encoding="utf-8")

        self.log.info("Saved response body: %s", body_path)
        self.log.info("Saved meta json: %s", meta_path)
        return {"body": body_path, "meta_json": meta_path}


# ----------------------------
# Parse cgi_info.js for addCfg(...) calls
# ----------------------------
def _iter_addcfg_args_str(js_text: str) -> List[str]:
    """
    Scan JavaScript text and extract the raw argument string inside addCfg(...).

    Algorithm (as specified):
      - Find literal substring: 'addCfg('
      - After that position, find the first occurrence of either:
          1) ');' or
          2) ')\\n'
        (No need to consider occurrences inside quotes.)
      - Use that ')' position as the end of the addCfg(...) call.
      - Extract args_str as the substring between '(' and ')'.

    Returns:
      A list of args_str strings (raw, not bracketed).
    """
    results: List[str] = []
    needle = "addCfg("
    i = 0

    while True:
        start = js_text.find(needle, i)
        if start < 0:
            break

        args_start = start + len(needle)

        end1 = js_text.find(");", args_start)   # points at ')' when found
        end2 = js_text.find(")\n", args_start)  # points at ')' when found

        ends = [e for e in (end1, end2) if e >= 0]
        if not ends:
            # No terminator found; stop scanning to avoid infinite loop.
            break

        end = min(ends)
        args_str = js_text[args_start:end]
        results.append(args_str)

        # Advance scanning position: move past the ')' we used.
        i = end + 1

    return results


def extract_wan_dhcp_dns_tokens(cgi_info_js_text: str) -> List[List[str]]:
    """
    Find addCfg(...) calls, parse args as JSON list using:
        args = json.loads("[" + args_str + "]")

    Then:
      - Find entries where:
            args[0] == "wan_dhcp_dns"
            args[1] == "ARC_WAN_0_IP4_DNS"
      - For each match, split args[2] by whitespace and log the tokens.

    Returns:
      List of token lists (one per match).
    """
    log = logging.getLogger("cgi_info_parser")

    args_str_list = _iter_addcfg_args_str(cgi_info_js_text)
    log.info("Found %d addCfg(...) call(s) by scanning", len(args_str_list))

    all_tokens: List[List[str]] = []

    for idx, args_str in enumerate(args_str_list, 1):
        try:
            args = json.loads("[" + args_str + "]")
        except Exception as e:
            log.debug("Skipping addCfg call #%d: JSON parse failed: %s; args_str head=%r", idx, e, args_str[:120])
            continue

        if not isinstance(args, list) or len(args) < 3:
            continue
        if args[0] != "wan_dhcp_dns" or args[1] != "ARC_WAN_0_IP4_DNS":
            continue
        if not isinstance(args[2], str):
            continue

        tokens = [t for t in args[2].split() if t]
        log.info('Match %d: args[0]=%r args[1]=%r args[2]=%r', idx, args[0], args[1], args[2])
        log.info("Match %d tokens (%d): %s", idx, len(tokens), tokens)
        all_tokens.append(tokens)

    log.info(
        'Matched %d call(s) where args[0]=="wan_dhcp_dns" and args[1]=="ARC_WAN_0_IP4_DNS"',
        len(all_tokens),
    )
    return all_tokens


# ----------------------------
# HTTP client (Referer management)
# ----------------------------
class HttpClient:
    def __init__(self, session: aiohttp.ClientSession, saver: ResponseSaver) -> None:
        self.session = session
        self.saver = saver
        self.last_url: Optional[str] = None
        self.log = logging.getLogger("request")

    async def request_text(
        self,
        method: str,
        url: str,
        *,
        data: Optional[Dict[str, Any]] = None,
    ) -> str:
        text, _content_type, _status = await self.request_text_meta(method, url, data=data)
        return text

    async def request_text_meta(
        self,
        method: str,
        url: str,
        *,
        data: Optional[Dict[str, Any]] = None,
    ) -> Tuple[str, str, int]:
        """
        Same behavior as request_text, but also returns (text, content_type, status).
        """
        method_u = method.upper()

        if data is not None:
            self.log.debug("%s %s data=%s", method_u, url, data)
        else:
            self.log.debug("%s %s", method_u, url)

        request_body_bytes = build_request_body_bytes(data)

        # Per-request header: Referer is the previous URL (omit for the first request)
        req_headers_override: Dict[str, str] = {}
        if self.last_url:
            req_headers_override["Referer"] = self.last_url

        # POST: explicitly set Content-Type and send urlencoded bytes
        req_data_to_send: Any = None
        if method_u == "POST":
            req_headers_override["Content-Type"] = "application/x-www-form-urlencoded"
            req_data_to_send = request_body_bytes

        async with self.session.request(
            method=method_u,
            url=url,
            data=req_data_to_send,
            headers=req_headers_override if req_headers_override else None,
        ) as resp:
            sent_req_headers = resp.request_info.headers
            response_body = await resp.read()

            content_type = resp.headers.get("Content-Type", "")

            self.saver.save(
                method=method_u,
                url=url,
                status=resp.status,
                reason=resp.reason or "",
                content_type=content_type,
                request_headers=sent_req_headers,
                request_body=request_body_bytes,
                request_data=data,
                response_headers=resp.headers,
                response_raw_headers=resp.raw_headers,
                response_body=response_body,
            )

            self.log.debug("%s %s -> %s, bytes=%d", method_u, url, resp.status, len(response_body))

            charset = resp.charset or "utf-8"
            text = response_body.decode(charset, errors="replace")
            self.log.debug("Response head: %r", text[:300])

            # Update Referer for the next request
            self.last_url = url

            resp.raise_for_status()
            return text, content_type, resp.status


# ----------------------------
# Flow implementation
# ----------------------------
async def login(client: HttpClient, cfg: Config) -> None:
    log = logging.getLogger("login")

    login_html_url = urljoin(cfg.base_url + "/", "login.html")
    html_text = await client.request_text("GET", login_html_url)

    httoken = get_httoken(html_text)
    pws = md5_hex(cfg.password)

    login_cgi_url = urljoin(cfg.base_url + "/", "login.cgi")
    form = {
        "name": cfg.name,
        "pws": pws,
        "url": "/",
        "mobile": "0",
        "httoken": httoken,
    }

    log.info("Logging in as name=%s", cfg.name)
    login_resp_html, _ct, _st = await client.request_text_meta("POST", login_cgi_url, data=form)

    # Rule: If login.cgi <title> is not "BUFFALO AirStation", treat as not logged in.
    assert_title_equals(login_resp_html, "BUFFALO AirStation", context="login.cgi response title check")

    log.info("Login POST completed")


async def fetch_info(client: HttpClient, cfg: Config) -> None:
    """
    Behavior:
      - Fetch /info.html and extract httoken
      - Rule: If info.html contains <title>, treat as abnormal.
      - Fetch /cgi/cgi_info.js?_tn={httoken} and save it (request_text already saves)
      - Rule: If cgi_info.js Content-Type is not application/x-javascript, treat as abnormal.
      - Additionally:
          - Scan for addCfg(...) calls
          - Parse args via json.loads("[" + args_str + "]")
          - If args[0]=="wan_dhcp_dns" and args[1]=="ARC_WAN_0_IP4_DNS",
            split args[2] by whitespace and log the resulting tokens
    """
    log = logging.getLogger("fetch_info")

    info_url = urljoin(cfg.base_url + "/", "info.html")
    html_text = await client.request_text("GET", info_url)

    # Rule: info.html must NOT contain <title>
    assert_title_absent(html_text, context="info.html title absence check")

    httoken = get_httoken(html_text)
    log.info("Extracted httoken from info.html: %r", httoken)

    cgi_info_url = urljoin(cfg.base_url + "/", f"cgi/cgi_info.js?_tn={httoken}")
    cgi_info_js_text, cgi_ct, _st = await client.request_text_meta("GET", cgi_info_url)

    # Rule: cgi_info.js Content-Type must be application/x-javascript
    assert_content_type_equals(cgi_ct, "application/x-javascript", context="cgi_info.js Content-Type check")

    extract_wan_dhcp_dns_tokens(cgi_info_js_text)


async def logout(client: HttpClient, cfg: Config) -> None:
    log = logging.getLogger("logout")

    logout_html_url = urljoin(cfg.base_url + "/", "logout.html")
    html_text = await client.request_text("GET", logout_html_url)

    httoken = get_httoken(html_text)
    pws = md5_hex(cfg.password)

    logout_cgi_url = urljoin(cfg.base_url + "/", "logout.cgi")
    form = {
        "name": cfg.name,
        "pws": pws,
        "url": "/",
        "mobile": "0",
        "httoken": httoken,
    }

    log.info("Logging out as name=%s", cfg.name)
    logout_resp_html, _ct, _st = await client.request_text_meta("POST", logout_cgi_url, data=form)

    # Rule: logout.cgi <title> must be "LOGIN"
    assert_title_equals(logout_resp_html, "LOGIN", context="logout.cgi response title check")

    log.info("Logout POST completed")


# ----------------------------
# main
# ----------------------------
async def main() -> None:
    setup_logging()

    # You can override the config path with environment variable CONFIG_PATH.
    # Default is ./config.toml
    config_path = Path(os.environ.get("CONFIG_PATH", "config.toml"))
    cfg = Config.from_toml(config_path)

    run_ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
    base_out = Path(os.environ.get("OUT_DIR", ".")) / "outputs"
    responses_dir = base_out / f"responses_{run_ts}"
    saver = ResponseSaver(responses_dir)

    jar = aiohttp.CookieJar(unsafe=True)

    headers = {
        "User-Agent": "aiohttp-client/1.0",
        "Cookie": cfg.cookie,
    }

    timeout = aiohttp.ClientTimeout(total=60)

    logger.info("Loaded config from: %s", config_path)
    logger.info("Base URL: %s", cfg.base_url)
    logger.info("Responses will be saved under: %s", responses_dir)

    async with aiohttp.ClientSession(cookie_jar=jar, headers=headers, timeout=timeout) as session:
        client = HttpClient(session, saver)

        logged_in = False
        try:
            await login(client, cfg)
            logged_in = True

            await fetch_info(client, cfg)

        finally:
            if logged_in:
                try:
                    await logout(client, cfg)
                except Exception:
                    logger.exception("Logout failed (ignored)")

    logger.info("Done")


if __name__ == "__main__":
    asyncio.run(main())
```
