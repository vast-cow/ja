---
title: "pipを使わずにwhlをダウンロードして展開する"
description: "curlやwgetでダウンロードし、unzipやPythonのzipfileで展開する手順を紹介。"
pubDatetime: 2026-10-20T12:44:00+09:00
---

`pip` を使わずに `.whl` ファイルをダウンロードして展開するには、`curl` または `wget` でダウンロードし、`unzip` または Python 標準ライブラリの `zipfile` で展開します。

`.whl` は ZIP 形式のファイルなので、展開自体に `pip` は不要です。

## 1. whlファイルをダウンロードする

PyPI のパッケージページから `.whl` のダウンロード URL を取得します。

例：https://pypi.org/project/requests/#files

**curlを使用する場合**

```bash
curl -L -o package.whl "https://files.pythonhosted.org/.../package.whl"
```

**wgetを使用する場合**

```bash
wget -O package.whl "https://files.pythonhosted.org/.../package.whl"
```

※ URL は実際の wheel ファイルの URL に置き換えてください。

## 2. whlファイルを展開する

### 方法A：unzipを使う

```bash
mkdir extracted
unzip package.whl -d extracted/
```

### 方法B：Python標準ライブラリを使う

```python
import zipfile

with zipfile.ZipFile("package.whl") as z:
    z.extractall("extracted")
```

## 3. PyPI APIを使ってダウンロードから展開まで自動化する

`pip` や外部コマンドを使用せず、Python 標準ライブラリだけで実現できます。

```python
import json
import urllib.request
import zipfile
from pathlib import Path

package = "requests"
version = "2.32.3"

# PyPI APIからパッケージ情報を取得
url = f"https://pypi.org/pypi/{package}/{version}/json"

with urllib.request.urlopen(url) as response:
    data = json.load(response)

# wheelファイルを取得
wheel = next(
    f for f in data["urls"]
    if f["filename"].endswith(".whl")
)

wheel_path = Path(wheel["filename"])

# ダウンロード
urllib.request.urlretrieve(wheel["url"], wheel_path)

# 展開
output_dir = Path("extracted")
output_dir.mkdir(exist_ok=True)

with zipfile.ZipFile(wheel_path) as z:
    z.extractall(output_dir)

print(f"Downloaded: {wheel_path}")
print(f"Extracted: {output_dir}")
```

このコードは、対象バージョンの最初の wheel を取得します。複数の wheel が存在するパッケージでは、Python バージョン・OS・CPU アーキテクチャとの互換性を別途確認する必要があります。

## 4. 展開後のディレクトリ構造

例えば `requests` の場合、以下のようになります。

```text
extracted/
├── requests/
│   ├── __init__.py
│   ├── api.py
│   ├── sessions.py
│   └── ...
└── requests-2.32.3.dist-info/
    ├── METADATA
    ├── WHEEL
    └── RECORD
```

展開したパッケージを import したい場合は、次のようにします。

```python
import sys

sys.path.insert(0, "extracted")

import requests
```

ただし、`requests` が依存するライブラリも別途必要です。展開だけでは依存関係は自動的に解決されません。

## 5. 注意事項

- **依存関係：** wheel の展開だけでは依存パッケージはインストールされません。
- **互換性：** C拡張を含む wheel では、Python バージョン・OS・CPU アーキテクチャとの互換性が必要です。
- **インストール処理：** ZIP 展開だけでは、wheel の `.data` ディレクトリの配置や実行スクリプトの生成など、通常のインストール処理は実行されません。
- **セキュリティ：** 信頼できる配布元から取得し、必要に応じてファイルのハッシュを検証してください。

## まとめ

| 目的 | 方法 |
|---|---|
| whlをダウンロード | `curl` / `wget` |
| whlを展開 | `unzip` |
| Pythonだけで展開 | `zipfile` |
| PyPIから自動取得 | `urllib.request` + PyPI API |
| 展開したパッケージを参照 | `sys.path` に展開先を追加 |

**結論：** `.whl` は ZIP ファイルなので、`pip` を使わずにダウンロード・展開できます。ただし、展開と正式なインストールは異なるため、実行時の依存関係や互換性には注意が必要です。
