---
pubDatetime: 2026-05-01T22:48:41+09:00
title: "headless Chromeでytmusicapiに必要な情報を取得するスクリプト"
description: "このスクリプトは、YouTube Musicでytmusicapiを利用するために必要なリクエスト情報を取得するためのものです。既に起動しているChromeまたはEdgeに接続し、YouTube Musicへアクセスしたときに発生する通信を監視して、必要なヘッダー情報をJSONファイルとして保存しま…"
---

このスクリプトは、YouTube Musicで`ytmusicapi`を利用するために必要なリクエスト情報を取得するためのものです。既に起動しているChromeまたはEdgeに接続し、YouTube Musicへアクセスしたときに発生する通信を監視して、必要なヘッダー情報をJSONファイルとして保存します。

## 目的

`ytmusicapi`を使うには、YouTube Musicへアクセスするときの認証情報やヘッダー情報が必要になる場合があります。

このスクリプトは、ブラウザ上で実際にYouTube Musicへアクセスし、そのときに送信される特定のAPIリクエストを検出します。対象となるのは、次のURLへのリクエストです。

```text
https://music.youtube.com/youtubei/v1/browse
```

このリクエストに含まれるヘッダー情報を取得し、`matched_request_headers.json`というファイルに保存します。

## 何をしているか

このスクリプトは、主に次の処理を行います。

1. リモートデバッグポートで起動しているChromeまたはEdgeに接続する
2. 既存のタブを選択する
3. User-Agentを通常のChromeに見えるように調整する
4. YouTube Musicのアップロードライブラリページへ移動する
5. 通信を監視し、対象のAPIリクエストだけを抽出する
6. リクエストのURL、メソッド、種別、ヘッダーをJSONに保存する
7. 処理後、使用したタブを新しいタブページへ戻す

## 事前準備

このスクリプトを使うには、ChromeまたはEdgeをリモートデバッグポート付きで起動しておく必要があります。

例として、Chromeを次のように起動します。

```bash
chrome --remote-debugging-port=9222
```

環境によっては、実行ファイル名やパスを指定する必要があります。

また、Python側ではPlaywrightが必要です。

```bash
pip install playwright
playwright install
```

## 使い方

まず、リモートデバッグポート付きでChromeまたはEdgeを起動します。

次に、そのブラウザでYouTube Musicにログインしておきます。ログイン済みのブラウザに接続することで、実際のアカウント状態に基づいたリクエストを取得できます。

その後、スクリプトを実行します。

```bash
python script.py
```

実行すると、スクリプトは既存のブラウザに接続し、次のページへ移動します。

```text
https://music.youtube.com/library/uploads
```

ページ読み込み中に、条件に一致するリクエストが見つかると、その情報がコンソールに表示されます。最後に、結果が次のファイルへ保存されます。

```text
matched_request_headers.json
```

## 出力される内容

出力ファイルには、次のような情報が含まれます。

```json
{
  "effective_user_agent": "...",
  "matched_count": 1,
  "matched_requests": [
    {
      "url": "...",
      "url_without_query": "https://music.youtube.com/youtubei/v1/browse",
      "method": "POST",
      "resource_type": "xhr",
      "headers": {
        "...": "..."
      }
    }
  ]
}
```

`matched_requests`の中に、対象APIリクエストの詳細が入ります。特に`headers`の内容が、`ytmusicapi`で利用するための重要な情報になります。

## User-Agentを調整する理由

headless Chromeを使うと、User-Agentに`HeadlessChrome`という文字列が含まれることがあります。

このスクリプトでは、それを`Chrome`に置き換えます。これにより、通常のChromeブラウザに近い状態でリクエストを送信できます。

また、`acceptLanguage`には日本語環境向けの値が設定されています。

```text
ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7
```

## 対象リクエストの判定

スクリプトは、すべての通信を保存するわけではありません。

クエリパラメータを除いたURLが、次の値と完全に一致するものだけを対象にします。

```text
https://music.youtube.com/youtubei/v1/browse
```

そのため、不要な画像、CSS、JavaScript、その他のAPI通信は基本的に無視されます。

## 注意点

このスクリプトは、既存のブラウザに接続して動作します。そのため、ブラウザをスクリプト側で閉じることはありません。

処理後は、使用したタブを次のページへ戻します。

```text
chrome://new-tab-page
```

また、対象のリクエストが発生しない場合、`matched_count`は`0`になります。その場合は、YouTube Musicにログインしているか、対象ページが正しく読み込まれているかを確認してください。

## まとめ

このスクリプトは、YouTube Musicの通信をPlaywrightで監視し、`ytmusicapi`に必要なヘッダー情報を取得するための補助ツールです。

既にログイン済みのChromeまたはEdgeに接続して動作するため、ブラウザの実際のセッション情報を利用できます。取得した情報は`matched_request_headers.json`に保存され、後から`ytmusicapi`の設定や認証情報の準備に利用できます。

```python
import json
import urllib.request
from urllib.parse import urlparse
from playwright.sync_api import sync_playwright, Request, Page


# ===== Configuration =====

CDP_ENDPOINT = "http://127.0.0.1:9222"  # Existing browser CDP port
URL_A = "https://music.youtube.com/library/uploads"

# Condition B:
# Match requests whose URL, excluding query parameters, is exactly:
# https://music.youtube.com/youtubei/v1/browse
URL_B_BASE = "https://music.youtube.com/youtubei/v1/browse"

OUTPUT_JSON = "matched_request_headers.json"


def assert_cdp_available(endpoint: str) -> None:
    version_url = endpoint.rstrip("/") + "/json/version"

    try:
        with urllib.request.urlopen(version_url, timeout=3) as res:
            if res.status != 200:
                raise RuntimeError(f"CDP endpoint returned HTTP {res.status}")
    except Exception as e:
        raise RuntimeError(
            f"CDP endpoint is not available: {version_url}\n"
            f"Please start Chrome/Edge with --remote-debugging-port=9222.\n"
            f"Original error: {e}"
        ) from e


def get_url_without_query(url: str) -> str:
    """
    Return the URL without query parameters or fragments.

    Example:
      https://music.youtube.com/youtubei/v1/browse?key=abc
      -> https://music.youtube.com/youtubei/v1/browse
    """
    parsed = urlparse(url)
    return f"{parsed.scheme}://{parsed.netloc}{parsed.path}"


def matches_condition_b(url: str) -> bool:
    """
    Condition B:
    Match only when the URL without query parameters is exactly URL_B_BASE.
    """
    return get_url_without_query(url) == URL_B_BASE


def pick_existing_page(context) -> Page:
    """
    Pick one existing tab.

    Prefer an chrome://new-tab-page tab if available.
    Otherwise, use the first existing tab.
    """
    pages = context.pages

    if not pages:
        raise RuntimeError(
            "No existing tab was found. Please open at least one tab in the CDP-connected browser."
        )

    for page in pages:
        if page.url == "chrome://new-tab-page":
            return page

    return pages[0]


def get_default_user_agent(page: Page) -> str:
    """
    Read navigator.userAgent from the current page environment.

    Using the actual User-Agent from the CDP-connected browser avoids mismatch
    between the real Chrome version and the User-Agent string.
    """
    user_agent = page.evaluate("navigator.userAgent")

    if not isinstance(user_agent, str) or not user_agent.strip():
        raise RuntimeError("Failed to read navigator.userAgent")

    return user_agent


def normalize_user_agent(user_agent: str) -> str:
    """
    Replace HeadlessChrome with Chrome.

    If the User-Agent does not contain HeadlessChrome, return it unchanged.
    """
    return user_agent.replace("HeadlessChrome", "Chrome")


def get_default_platform(page: Page) -> str:
    """
    Read navigator.platform from the current page environment.

    Return an empty string if it cannot be read.
    """
    try:
        platform = page.evaluate("navigator.platform")
    except Exception:
        return ""

    if not isinstance(platform, str):
        return ""

    return platform


def spoof_user_agent(context, page: Page) -> str:
    """
    Override the User-Agent for an existing CDP-connected browser page.

    For an existing browser connected through CDP, use CDP's
    Network.setUserAgentOverride instead of new_context(user_agent=...).

    Steps:
      1. Read the current navigator.userAgent.
      2. Replace HeadlessChrome with Chrome.
      3. Apply the value using Network.setUserAgentOverride.

    Returns:
      The actual User-Agent value that was applied.
    """
    default_user_agent = get_default_user_agent(page)
    override_user_agent = normalize_user_agent(default_user_agent)
    platform = get_default_platform(page)

    print(f"Default User-Agent : {default_user_agent}")
    print(f"Override User-Agent: {override_user_agent}")
    print(f"Platform           : {platform}")

    cdp_session = context.new_cdp_session(page)

    cdp_session.send("Network.enable")

    params = {
        "userAgent": override_user_agent,
        "acceptLanguage": "ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7",
    }

    if platform:
        params["platform"] = platform

    cdp_session.send("Network.setUserAgentOverride", params)

    return override_user_agent


def main() -> None:
    matched_requests = []

    assert_cdp_available(CDP_ENDPOINT)

    with sync_playwright() as p:
        browser = p.chromium.connect_over_cdp(CDP_ENDPOINT)

        if not browser.contexts:
            raise RuntimeError("No existing browser context was found.")

        context = browser.contexts[0]
        page = pick_existing_page(context)

        # Apply User-Agent spoofing
        effective_user_agent = spoof_user_agent(context, page)

        def handle_request(request: Request) -> None:
            url = request.url

            if not matches_condition_b(url):
                return

            try:
                headers = request.all_headers()

                record = {
                    "url": url,
                    "url_without_query": get_url_without_query(url),
                    "method": request.method,
                    "resource_type": request.resource_type,
                    "headers": headers,
                }

                matched_requests.append(record)

                print("=== MATCHED REQUEST ===")
                print(json.dumps(record, ensure_ascii=False, indent=2))

            except Exception as e:
                print(f"[WARN] Failed to read headers for {url}: {e}")

        context.on("request", handle_request)

        try:
            # Navigate the existing tab to URL A
            page.goto(URL_A, wait_until="domcontentloaded")

            try:
                page.wait_for_load_state("networkidle", timeout=15_000)
            except Exception:
                pass

            output = {
                "effective_user_agent": effective_user_agent,
                "matched_count": len(matched_requests),
                "matched_requests": matched_requests,
            }

            with open(OUTPUT_JSON, "w", encoding="utf-8") as f:
                json.dump(output, f, ensure_ascii=False, indent=2)

            print(f"Saved: {OUTPUT_JSON}")
            print(f"Matched count: {len(matched_requests)}")

        finally:
            # After collection, return the existing tab to the new tab page
            try:
                page.goto("chrome://new-tab-page", wait_until="domcontentloaded")
            except Exception as e:
                print(f"[WARN] Failed to navigate tab to chrome://new-tab-page: {e}")

            # Do not close the existing browser
            # browser.close()


if __name__ == "__main__":
    main()
```
