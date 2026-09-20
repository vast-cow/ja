---
pubDatetime: 2026-06-18T13:45:52+09:00
title: "WebArchive と MHTML ファイルを相互変換するシンプルなツール"
description: "目的 このツールは、Apple の .webarchive 形式と MHTML/MHT 形式の間でファイルを変換します。 ある形式で保存した Web ページを、別の形式で開いたり、共有したり、保存したりする必要がある場合に便利です。たとえば、Safari は .webarchive を使用することが…"
---

## 目的

このツールは、Apple の `.webarchive` 形式と MHTML/MHT 形式の間でファイルを変換します。

ある形式で保存した Web ページを、別の形式で開いたり、共有したり、保存したりする必要がある場合に便利です。たとえば、Safari は `.webarchive` を使用することが多く、一方で多くのブラウザーやメールベースのツールは `.mhtml` や `.mht` を使用します。

このコンバーターは双方向の変換に対応しています。

* `.mhtml` または `.mht` → `.webarchive`
* `.webarchive` → `.mhtml`

コマンドラインから簡単に実行できるよう設計されており、外部の Python パッケージは必要ありません。

## 主な利点

このツールにより、以下のことが可能になります。

* 保存済み Web ページを一般的なアーカイブ形式間で変換できる
* メインページと関連リソースをまとめて保持できる
* 必要に応じて出力ファイル名を指定できる
* 変換方向を自動的に判別できる
* 特殊なケース向けのオプション設定を利用できる

Python の標準ライブラリのみを使用するため、セットアップは最小限で済みます。

## 必要条件

このツールを使用するには、以下が必要です。

* Python 3.9 以降
* `webarchive_mhtml_converter.py` スクリプト
* 変換対象の `.webarchive`、`.mhtml`、または `.mht` ファイル

追加ライブラリのインストールは不要です。

## 基本的な使い方

最も簡単な使い方は、入力ファイルを指定するだけです。

MHTML ファイルの場合:

```bash
python webarchive_mhtml_converter.py page.mhtml
```

次のファイルが作成されます。

```bash
page.webarchive
```

WebArchive ファイルの場合:

```bash
python webarchive_mhtml_converter.py page.webarchive
```

次のファイルが作成されます。

```bash
page.mhtml
```

ツールは拡張子から入力ファイル形式を判別し、自動的に反対形式へ変換します。

## 出力ファイル名を指定する

`-o` または `--output` を使用して出力パスを指定できます。

例:

```bash
python webarchive_mhtml_converter.py page.mhtml -o converted.webarchive
```

別の例:

```bash
python webarchive_mhtml_converter.py page.webarchive -o converted.mhtml
```

元のファイル名をそのまま残したい場合や、別のフォルダーに保存したい場合に便利です。

## 出力形式を手動で指定する

入力ファイルの拡張子が不明確な場合は、`--to` オプションで変換先形式を指定できます。

WebArchive ファイルを作成する場合:

```bash
python webarchive_mhtml_converter.py page.dat --to webarchive -o page.webarchive
```

MHTML ファイルを作成する場合:

```bash
python webarchive_mhtml_converter.py page.dat --to mhtml -o page.mhtml
```

これにより、ツールが変換方向を自動判別できない場合でも混乱を避けられます。

## オプション設定

### CID CSS リンクを保持する

MHTML から WebArchive へ変換する際、ツールは通常、互換性向上のために特定のスタイルシートリンクを調整します。

これらのリンクを変更せずに保持したい場合は、以下を使用します。

```bash
python webarchive_mhtml_converter.py page.mhtml --keep-cid-css-link
```

### WebArchive の plist 形式を選択する

MHTML から WebArchive へ変換する際、デフォルトの出力形式はバイナリです。

代わりに XML plist を生成するには、以下を使用します。

```bash
python webarchive_mhtml_converter.py page.mhtml --plist-format xml
```

### サブフレームを除外する

WebArchive から MHTML へ変換する際、通常はサブフレームのアーカイブも含まれます。

これらを除外するには、以下を使用します。

```bash
python webarchive_mhtml_converter.py page.webarchive --no-subframes
```

## エラーハンドリング

ファイルを変換できない場合、ツールはエラーメッセージを表示します。

主な原因としては以下があります。

* 入力ファイルが存在しない
* 入力パスがファイルではない
* ファイル拡張子が指定した出力形式と一致しない
* 変換方向を判別できない
* 変換方向に対して無効なオプションが使用されている

これらのメッセージにより、再実行前に修正すべき問題を特定できます。

## まとめ

`webarchive_mhtml_converter.py` は、Apple WebArchive 形式と MHTML/MHT 形式の間で保存済み Web ページを変換するための小型コマンドラインツールです。

異なるブラウザー、システム、またはワークフロー間でアーカイブ済み Web ページを扱う実用的な方法を必要とするユーザーに適しています。基本的な変換は 1 つのコマンドで実行でき、必要に応じてオプション設定によりより細かな制御も可能です。

```python
#!/usr/bin/env python3
"""
Unified converter for Apple .webarchive and MHTML/MHT files.

This combines the behavior of the two separate tools:
  - MHTML/MHT -> Apple WebArchive
  - Apple WebArchive -> MHTML/MHT

Requirements:
  - Python 3.9+
  - No external dependencies; uses only the Python standard library.

Examples:
  # Auto-detect direction from input extension
  python webarchive_mhtml_converter.py page.mhtml
  python webarchive_mhtml_converter.py page.webarchive

  # Explicit output path
  python webarchive_mhtml_converter.py page.mhtml -o page.webarchive
  python webarchive_mhtml_converter.py page.webarchive -o page.mhtml

  # Explicit conversion target
  python webarchive_mhtml_converter.py page.dat --to webarchive -o page.webarchive
  python webarchive_mhtml_converter.py page.dat --to mhtml -o page.mhtml

  # Direction-specific options
  python webarchive_mhtml_converter.py page.mhtml --keep-cid-css-link
  python webarchive_mhtml_converter.py page.webarchive --no-subframes
"""

from __future__ import annotations

import argparse
import base64
import email
import mimetypes
import plistlib
import re
import sys
import uuid
from email import policy
from email.message import EmailMessage, Message
from email.parser import BytesParser
from pathlib import Path
from typing import Any, Dict, Iterable, List, Optional, Sequence, Tuple
from urllib.parse import quote, urljoin, urlparse


Resource = Dict[str, Any]
Archive = Dict[str, Any]

MHTML_SUFFIXES = {".mhtml", ".mht"}
WEBARCHIVE_SUFFIXES = {".webarchive"}

TEXT_LIKE_MIME_PREFIXES = ("text/",)
TEXT_LIKE_MIME_TYPES = {
    "application/javascript",
    "application/ecmascript",
    "application/json",
    "application/xml",
    "application/xhtml+xml",
    "image/svg+xml",
}

DEFAULT_BINARY_MIME = "application/octet-stream"


# ---------------------------------------------------------------------------
# Shared helpers
# ---------------------------------------------------------------------------


def normalize_mime_type(mime_type: str) -> str:
    return (mime_type or DEFAULT_BINARY_MIME).split(";", 1)[0].strip().lower()


def is_text_like_mime(mime_type: str) -> bool:
    mt = normalize_mime_type(mime_type)
    return mt.startswith(TEXT_LIKE_MIME_PREFIXES) or mt in TEXT_LIKE_MIME_TYPES


def kind_from_suffix(path: Path) -> Optional[str]:
    suffix = path.suffix.lower()
    if suffix in WEBARCHIVE_SUFFIXES:
        return "webarchive"
    if suffix in MHTML_SUFFIXES:
        return "mhtml"
    return None


def default_output_path(input_path: Path, target: str) -> Path:
    if target == "mhtml":
        return input_path.with_suffix(".mhtml")
    if target == "webarchive":
        return input_path.with_suffix(".webarchive")
    raise ValueError(f"Unsupported target: {target}")


def infer_target(input_path: Path, output_path: Optional[Path], requested_target: str) -> str:
    """
    Return the output target: "mhtml" or "webarchive".

    In auto mode, the input extension is authoritative. If the input extension is
    unknown, the output extension is used as a fallback.
    """
    if requested_target != "auto":
        return requested_target

    input_kind = kind_from_suffix(input_path)
    if input_kind == "webarchive":
        return "mhtml"
    if input_kind == "mhtml":
        return "webarchive"

    if output_path is not None:
        output_kind = kind_from_suffix(output_path)
        if output_kind in {"mhtml", "webarchive"}:
            return output_kind

    raise ValueError(
        "Could not infer conversion direction. Use --to mhtml or --to webarchive."
    )


def validate_paths_and_direction(
    input_path: Path,
    output_path: Path,
    target: str,
    *,
    requested_target: str,
) -> None:
    if not input_path.exists():
        raise FileNotFoundError(f"Input file does not exist: {input_path}")
    if not input_path.is_file():
        raise ValueError(f"Input path is not a file: {input_path}")

    input_kind = kind_from_suffix(input_path)
    output_kind = kind_from_suffix(output_path)

    if input_kind == target:
        raise ValueError(
            f"Input extension already looks like the requested output type ({target}). "
            "Check --to or the input file extension."
        )

    if output_kind is not None and output_kind != target:
        raise ValueError(
            f"Output extension implies {output_kind}, but target is {target}: {output_path}"
        )

    if requested_target == "auto" and input_kind is None and output_kind is None:
        raise ValueError(
            "Could not infer conversion direction from extensions. Use --to explicitly."
        )


# ---------------------------------------------------------------------------
# MHTML/MHT -> Apple WebArchive
# ---------------------------------------------------------------------------


def normalize_cid(value: Optional[str]) -> Optional[str]:
    if not value:
        return None
    value = value.strip()
    if value.startswith("<") and value.endswith(">"):
        value = value[1:-1]
    return value


def cid_url_from_part(part: EmailMessage) -> Optional[str]:
    cid = normalize_cid(part.get("Content-ID"))
    return f"cid:{cid}" if cid else None


def is_absolute_or_cid(url: str) -> bool:
    if url.startswith("cid:"):
        return True
    parsed = urlparse(url)
    return bool(parsed.scheme)


def text_encoding_from_content_type(content_type: str) -> str:
    msg = Message()
    msg["content-type"] = content_type
    return msg.get_content_charset() or "utf-8"


def fix_mime_type(url: str, mime: str) -> str:
    mime = normalize_mime_type(mime)
    path = urlparse(url).path.lower()
    guessed, _ = mimetypes.guess_type(path)

    if path.endswith(".css") and mime not in {"text/css", DEFAULT_BINARY_MIME}:
        return "text/css"
    if path.endswith(".css") and mime == DEFAULT_BINARY_MIME:
        return "text/css"
    if path.endswith(".js") and mime in {"text/plain", DEFAULT_BINARY_MIME}:
        return "application/javascript"
    if guessed and mime == DEFAULT_BINARY_MIME:
        return normalize_mime_type(guessed)
    return mime


def payload_bytes(part: EmailMessage) -> bytes:
    data = part.get_payload(decode=True)
    if data is not None:
        return data

    raw = part.get_payload()
    if isinstance(raw, str):
        enc = part.get_content_charset() or "utf-8"
        return raw.encode(enc, errors="replace")

    return b""


def choose_root_part(msg: EmailMessage, parts: List[EmailMessage]) -> EmailMessage:
    start = normalize_cid(msg.get_param("start"))
    if start:
        for part in parts:
            if normalize_cid(part.get("Content-ID")) == start:
                return part

    # RFC-compatible fallback for typical MHTML saved by browsers.
    for part in parts:
        if part.get_content_type() in {"text/html", "application/xhtml+xml"}:
            return part

    raise ValueError("No HTML root part found in MHTML")


def mhtml_part_url(part: EmailMessage, base_url: Optional[str], index: int) -> str:
    loc = (part.get("Content-Location") or "").strip()
    if loc:
        if is_absolute_or_cid(loc):
            return loc
        if base_url:
            return urljoin(base_url, loc)
        return loc

    cid = cid_url_from_part(part)
    if cid:
        return cid

    return f"mhtml-resource-{index}"


def make_web_resource(url: str, mime: str, data: bytes, *, content_type_header: Optional[str] = None) -> Resource:
    fixed_mime = fix_mime_type(url, mime)

    if is_text_like_mime(fixed_mime):
        data = data.rstrip(b"\x00")

    resource: Resource = {
        "WebResourceURL": url,
        "WebResourceMIMEType": fixed_mime,
        "WebResourceData": data,
    }

    if is_text_like_mime(fixed_mime):
        resource["WebResourceTextEncodingName"] = text_encoding_from_content_type(
            content_type_header or fixed_mime
        )

    return resource


def strip_inline_charset(css: str) -> str:
    # @charset is only meaningful as a stylesheet byte-stream marker; remove it for <style>.
    return re.sub(r'^\s*@charset\s+["\'][^"\']+["\']\s*;\s*', "", css, flags=re.I)


def replace_stylesheet_link(html: str, url: str, css_text: str) -> str:
    style_tag = '<style type="text/css">\n' + strip_inline_charset(css_text) + "\n</style>"

    def repl(match: re.Match[str]) -> str:
        tag = match.group(0)
        href_pat = r'\bhref\s*=\s*(["\'])' + re.escape(url) + r'\1'
        rel_pat = r'\brel\s*=\s*(["\'])[^"\']*stylesheet[^"\']*\1'
        if re.search(href_pat, tag, flags=re.I) and re.search(rel_pat, tag, flags=re.I):
            return style_tag
        return tag

    return re.sub(r"<link\b[^>]*>", repl, html, flags=re.I)


def inline_cid_stylesheets(archive: Archive) -> None:
    resources = [archive["WebMainResource"], *archive.get("WebSubresources", [])]

    cid_css: List[Tuple[str, str]] = []
    for resource in resources:
        if resource.get("WebResourceMIMEType") == "text/css" and str(
            resource.get("WebResourceURL", "")
        ).startswith("cid:"):
            enc = resource.get("WebResourceTextEncodingName") or "utf-8"
            css = (
                resource.get("WebResourceData", b"")
                .rstrip(b"\x00")
                .decode(enc, errors="replace")
            )
            cid_css.append((resource["WebResourceURL"], css))

    if not cid_css:
        return

    for resource in resources:
        if resource.get("WebResourceMIMEType") not in {"text/html", "application/xhtml+xml"}:
            continue
        enc = resource.get("WebResourceTextEncodingName") or "utf-8"
        html = (
            resource.get("WebResourceData", b"")
            .rstrip(b"\x00")
            .decode(enc, errors="replace")
        )
        for url, css in cid_css:
            html = replace_stylesheet_link(html, url, css)
        resource["WebResourceData"] = html.encode(enc, errors="replace")


def parse_mhtml(path: Path, *, inline_cid_css: bool = True) -> Archive:
    msg = BytesParser(policy=policy.default).parsebytes(path.read_bytes())
    if not msg.is_multipart():
        raise ValueError("Input is not multipart MHTML")

    parts = [p for p in msg.walk() if not p.is_multipart()]
    root = choose_root_part(msg, parts)

    snapshot_url = (msg.get("Snapshot-Content-Location") or "").strip() or None
    root_loc = (root.get("Content-Location") or "").strip() or snapshot_url
    root_url = root_loc or cid_url_from_part(root) or "about:blank"

    main_data = payload_bytes(root)
    main_mime = root.get_content_type() or "text/html"
    main_resource = make_web_resource(
        root_url,
        main_mime,
        main_data,
        content_type_header=root.get("Content-Type"),
    )

    subresources: List[Resource] = []
    seen_main_identity = id(root)

    for idx, part in enumerate(parts, start=1):
        if id(part) == seen_main_identity:
            # Add a cid: alias for the root only if the HTML might refer to it.
            cid = cid_url_from_part(part)
            if cid and cid != root_url:
                subresources.append(
                    make_web_resource(
                        cid,
                        main_mime,
                        main_data,
                        content_type_header=part.get("Content-Type"),
                    )
                )
            continue

        url = mhtml_part_url(part, root_url, idx)
        mime = part.get_content_type() or DEFAULT_BINARY_MIME
        data = payload_bytes(part)
        subresources.append(
            make_web_resource(
                url,
                mime,
                data,
                content_type_header=part.get("Content-Type"),
            )
        )

        # Alias Content-ID as cid:... when Content-Location differs.
        cid = cid_url_from_part(part)
        if cid and cid != url:
            subresources.append(
                make_web_resource(
                    cid,
                    mime,
                    data,
                    content_type_header=part.get("Content-Type"),
                )
            )

        # Keep raw relative Content-Location as an alias for snapshots that use it verbatim.
        raw_loc = (part.get("Content-Location") or "").strip()
        if raw_loc and raw_loc != url and not is_absolute_or_cid(raw_loc):
            subresources.append(
                make_web_resource(
                    raw_loc,
                    mime,
                    data,
                    content_type_header=part.get("Content-Type"),
                )
            )

    archive: Archive = {
        "WebMainResource": main_resource,
        "WebSubresources": subresources,
    }

    if inline_cid_css:
        inline_cid_stylesheets(archive)

    return archive


def convert_mhtml_to_webarchive(
    input_path: Path,
    output_path: Path,
    *,
    inline_cid_css: bool = True,
    plist_format: str = "binary",
) -> None:
    archive = parse_mhtml(input_path, inline_cid_css=inline_cid_css)
    fmt = plistlib.FMT_BINARY if plist_format == "binary" else plistlib.FMT_XML
    with output_path.open("wb") as f:
        plistlib.dump(archive, f, fmt=fmt, sort_keys=False)


# ---------------------------------------------------------------------------
# Apple WebArchive -> MHTML/MHT
# ---------------------------------------------------------------------------


def load_webarchive(path: Path) -> Archive:
    """
    Load .webarchive with plistlib.

    plistlib supports both XML plist and binary plist. In .webarchive files,
    WebResourceData is preserved as bytes.
    """
    with path.open("rb") as f:
        archive = plistlib.load(f)

    if not isinstance(archive, dict):
        raise ValueError("Invalid webarchive: root object is not a dictionary")

    if not isinstance(archive.get("WebMainResource"), dict):
        raise ValueError("Invalid webarchive: missing WebMainResource")

    return archive


def decode_webresource_data(value: Any) -> bytes:
    """
    Decode WebResourceData.

    With plistlib this should normally be bytes. Other forms are handled
    defensively for compatibility with unusual plist representations.
    """
    if value is None:
        return b""

    if isinstance(value, bytes):
        return value

    if isinstance(value, bytearray):
        return bytes(value)

    if isinstance(value, str):
        s = value.strip()

        # Defensive support for base64 strings.
        try:
            padded = s + ("=" * ((4 - len(s) % 4) % 4))
            decoded = base64.b64decode(padded, validate=True)
            if base64.b64encode(decoded).decode("ascii").rstrip("=") == s.rstrip("="):
                return decoded
        except Exception:
            pass

        # Defensive support for plain text strings.
        return value.encode("utf-8")

    if isinstance(value, dict):
        for key in ("bytes", "data", "base64", "WebResourceData"):
            if key in value:
                return decode_webresource_data(value[key])

    raise TypeError(f"Unsupported WebResourceData type: {type(value).__name__}")


def guess_mime_type(resource: Resource, data: bytes, is_main: bool) -> str:
    mime = resource.get("WebResourceMIMEType")
    if isinstance(mime, str) and mime.strip():
        return normalize_mime_type(mime)

    url = resource.get("WebResourceURL")
    if isinstance(url, str):
        guessed, _ = mimetypes.guess_type(url)
        if guessed:
            return normalize_mime_type(guessed)

    if is_main:
        return "text/html"

    # Basic signature detection.
    if data.startswith(b"\x89PNG\r\n\x1a\n"):
        return "image/png"
    if data.startswith(b"\xff\xd8\xff"):
        return "image/jpeg"
    if data.startswith(b"GIF87a") or data.startswith(b"GIF89a"):
        return "image/gif"
    if data.startswith(b"RIFF") and data[8:12] == b"WEBP":
        return "image/webp"
    if data.startswith(b"\x00\x00\x00") and b"ftypavif" in data[:32]:
        return "image/avif"
    if data.startswith(b"wOFF"):
        return "font/woff"
    if data.startswith(b"wOF2"):
        return "font/woff2"
    if data.lstrip().startswith((b"<svg", b"<?xml")):
        return "image/svg+xml"

    return DEFAULT_BINARY_MIME


def get_charset(resource: Resource, mime_type: str) -> Optional[str]:
    encoding = resource.get("WebResourceTextEncodingName")
    if isinstance(encoding, str) and encoding.strip():
        return encoding.strip()

    if is_text_like_mime(mime_type):
        return "utf-8"

    return None


def content_location(resource: Resource, fallback: str) -> str:
    url = resource.get("WebResourceURL")
    if isinstance(url, str) and url.strip():
        return url.strip()
    return fallback


def sanitize_header_value(value: str) -> str:
    """
    Keep Content-Location browser-friendly.

    Do not use email.header/Header or RFC 2047 encoded-word for URLs. Remove
    CR/LF to prevent header injection. Percent-encode non-ASCII only.
    """
    value = value.replace("\r", "").replace("\n", "")

    try:
        value.encode("ascii")
        return value
    except UnicodeEncodeError:
        # Keep normal URL punctuation readable; encode non-ASCII characters.
        return quote(value, safe=":/?#[]@!$&'()*+,;=%")


def fold_header_line(name: str, value: str, limit: int = 998) -> bytes:
    """
    Fold long header lines without RFC 2047 encoding.

    RFC 5322 hard limit is 998 octets per line. MHTML readers are usually
    happier with raw folded URLs than encoded-word URLs.
    """
    prefix = f"{name}: "
    raw = sanitize_header_value(value)

    line = prefix + raw
    encoded = line.encode("utf-8")

    if len(encoded) <= limit:
        return encoded + b"\r\n"

    # Fold on safe URL boundary characters where possible.
    out: List[bytes] = []
    current = prefix

    for token in re.split(r"([/?&=#.;,:_-])", raw):
        if token == "":
            continue

        candidate = current + token
        if len(candidate.encode("utf-8")) <= limit:
            current = candidate
            continue

        out.append(current.encode("utf-8") + b"\r\n")
        current = " " + token

    if current:
        out.append(current.encode("utf-8") + b"\r\n")

    return b"".join(out)


def make_content_type(mime_type: str, charset: Optional[str]) -> str:
    if charset and is_text_like_mime(mime_type):
        return f'{mime_type}; charset="{charset}"'
    return mime_type


def wrap_base64(data: bytes) -> bytes:
    """Base64-wrap to MIME's conventional 76-character lines."""
    return base64.encodebytes(data).replace(b"\n", b"\r\n")


def make_mhtml_part(
    *,
    data: bytes,
    mime_type: str,
    charset: Optional[str],
    location: str,
    content_id: Optional[str] = None,
) -> bytes:
    headers = bytearray()
    headers += fold_header_line("Content-Type", make_content_type(mime_type, charset))
    headers += b"Content-Transfer-Encoding: base64\r\n"
    headers += fold_header_line("Content-Location", location)

    if content_id:
        headers += fold_header_line("Content-ID", f"<{content_id}>")

    return bytes(headers) + b"\r\n" + wrap_base64(data)


def iter_archive_resources(
    archive: Archive,
    *,
    include_subframes: bool,
    prefix: str = "",
) -> Iterable[Tuple[Resource, bool, str]]:
    """
    Yield (resource, is_main, fallback_location).

    Main resource is yielded before subresources for each archive. Subframe
    archives are recursively included after subresources.
    """
    main = archive.get("WebMainResource")
    if isinstance(main, dict):
        yield main, True, f"{prefix}main-resource"

    subresources = archive.get("WebSubresources") or archive.get("WebSubResources") or []
    if isinstance(subresources, list):
        for i, resource in enumerate(subresources, start=1):
            if isinstance(resource, dict):
                yield resource, False, f"{prefix}resource-{i}"

    if include_subframes:
        subframes = archive.get("WebSubframeArchives") or []
        if isinstance(subframes, list):
            for frame_index, subarchive in enumerate(subframes, start=1):
                if isinstance(subarchive, dict):
                    frame_prefix = f"{prefix}frame-{frame_index}-"
                    yield from iter_archive_resources(
                        subarchive,
                        include_subframes=True,
                        prefix=frame_prefix,
                    )


def build_mhtml(
    archive: Archive,
    *,
    source_name: str,
    include_subframes: bool = True,
) -> bytes:
    boundary = f"----=_NextPart_{uuid.uuid4().hex}"
    boundary_bytes = boundary.encode("ascii")

    resources = list(
        iter_archive_resources(
            archive,
            include_subframes=include_subframes,
        )
    )

    if not resources:
        raise ValueError("Invalid webarchive: no resources found")

    main_resource, _, _ = resources[0]
    main_data = decode_webresource_data(main_resource.get("WebResourceData"))
    main_mime = guess_mime_type(main_resource, main_data, is_main=True)

    lines = bytearray()
    lines += b"MIME-Version: 1.0\r\n"
    lines += fold_header_line("Subject", f"Converted from {source_name}")
    lines += fold_header_line(
        "Content-Type",
        f'multipart/related; type="{main_mime}"; start="<main-resource>"; boundary="{boundary}"',
    )
    lines += b"\r\n"
    lines += b"This is a multi-part message in MIME format.\r\n"

    seen_locations = set()

    for resource_index, (resource, is_main, fallback_location) in enumerate(resources):
        data = decode_webresource_data(resource.get("WebResourceData"))
        if not data:
            continue

        mime_type = guess_mime_type(resource, data, is_main=is_main)
        charset = get_charset(resource, mime_type)
        location = content_location(resource, fallback_location)

        if location in seen_locations:
            continue
        seen_locations.add(location)

        content_id = "main-resource" if resource_index == 0 else None

        lines += b"\r\n--" + boundary_bytes + b"\r\n"
        lines += make_mhtml_part(
            data=data,
            mime_type=mime_type,
            charset=charset,
            location=location,
            content_id=content_id,
        )

    lines += b"\r\n--" + boundary_bytes + b"--\r\n"
    return bytes(lines)


def convert_webarchive_to_mhtml(
    input_path: Path,
    output_path: Path,
    *,
    include_subframes: bool = True,
) -> None:
    archive = load_webarchive(input_path)
    mhtml = build_mhtml(
        archive,
        source_name=input_path.name,
        include_subframes=include_subframes,
    )
    output_path.write_bytes(mhtml)


# ---------------------------------------------------------------------------
# CLI
# ---------------------------------------------------------------------------


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description="Convert between Apple .webarchive and .mhtml/.mht files."
    )
    parser.add_argument("input", type=Path, help="Input .webarchive, .mhtml, or .mht file")
    parser.add_argument(
        "-o",
        "--output",
        type=Path,
        help="Output path. Default: input name with the opposite extension.",
    )
    parser.add_argument(
        "--to",
        choices=("auto", "mhtml", "webarchive"),
        default="auto",
        help=(
            "Output format. Default auto: .webarchive input becomes .mhtml; "
            ".mhtml/.mht input becomes .webarchive."
        ),
    )
    parser.add_argument(
        "--keep-cid-css-link",
        action="store_true",
        help=(
            "For MHTML -> WebArchive only: do not inline cid: stylesheet links. "
            "Default is to inline them for better iOS WebArchive compatibility."
        ),
    )
    parser.add_argument(
        "--plist-format",
        choices=("binary", "xml"),
        default="binary",
        help="For MHTML -> WebArchive only: plist output format. Default: binary.",
    )
    parser.add_argument(
        "--no-subframes",
        action="store_true",
        help="For WebArchive -> MHTML only: do not include WebSubframeArchives recursively.",
    )
    return parser


def run(args: argparse.Namespace) -> int:
    input_path: Path = args.input
    target = infer_target(input_path, args.output, args.to)
    output_path: Path = args.output or default_output_path(input_path, target)

    validate_paths_and_direction(
        input_path,
        output_path,
        target,
        requested_target=args.to,
    )

    if target == "webarchive":
        if args.no_subframes:
            raise ValueError("--no-subframes applies only to WebArchive -> MHTML conversion")
        convert_mhtml_to_webarchive(
            input_path,
            output_path,
            inline_cid_css=not args.keep_cid_css_link,
            plist_format=args.plist_format,
        )
    elif target == "mhtml":
        if args.keep_cid_css_link:
            raise ValueError("--keep-cid-css-link applies only to MHTML -> WebArchive conversion")
        if args.plist_format != "binary":
            raise ValueError("--plist-format applies only to MHTML -> WebArchive conversion")
        convert_webarchive_to_mhtml(
            input_path,
            output_path,
            include_subframes=not args.no_subframes,
        )
    else:
        raise ValueError(f"Unsupported target: {target}")

    print(f"wrote: {output_path}")
    return 0


def main(argv: Optional[Sequence[str]] = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    try:
        return run(args)
    except Exception as e:
        print(f"error: {e}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
```
