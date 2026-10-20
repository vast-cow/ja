---
title: "pipを使わずにwhlをダウンロードして展開する"
description: "curlやwgetでダウンロードし、unzipやPythonのzipfileで展開する手順を紹介。"
pubDatetime: 2026-10-20T12:44:00+09:00
---

`pip` を使わずに `.whl` ファイルをダウンロードして展開するには、`curl` または `wget` でダウンロードし、`unzip` または Python 標準ライブラリの `zipfile` で展開します。

`.whl` は実体として ZIP 形式のファイルなので、`pip` は不要です。

## 1. whl ファイルをダウンロードする

PyPI から取得する場合、まず以下にアクセスします。

[https://pypi.org/project/requests/#files](https://pypi.org/project/requests/?utm_source=chatgpt.com#files)

対象の `.whl` の URL を取得し、ダウンロードします。

```
curl -L -o package.whl "https://files.pythonhosted.org/.../package.whl"
```

または、

```
wget -O package.whl "https://files.pythonhosted.org/.../package.whl"
```

## 2. whl ファイルを展開する

### 方法A：unzip を使う

```
mkdir extracted
unzip package.whl -d extracted/
```

### 方法B：Python 標準ライブラリだけを使う

```
import zipfilewith zipfile.ZipFile("package.whl") as z:    z.extractall("extracted")
```

これで `.whl` の中身を展開できます。

## 3. PyPI API を使ってダウンロードから展開まで自動化する

`pip` も外部コマンドも使用せず、Python 標準ライブラリだけで実現できます。

```
import jsonimport urllib.requestimport zipfilefrom pathlib import Pathpackage = "requests"version = "2.32.3"# PyPI APIからパッケージ情報を取得url = f"https://pypi.org/pypi/{package}/{version}/json"with urllib.request.urlopen(url) as response:    data = json.load(response)# wheelファイルを取得wheel = next(    f for f in data["urls"]    if f["filename"].endswith(".whl"))wheel_path = Path(wheel["filename"])# ダウンロードurllib.request.urlretrieve(wheel["url"], wheel_path)# 展開output_dir = Path("extracted")output_dir.mkdir(exist_ok=True)with zipfile.ZipFile(wheel_path) as z:    z.extractall(output_dir)print(f"Downloaded: {wheel_path}")print(f"Extracted: {output_dir}")
```

## 4. 展開後のディレクトリ構造

例えば `requests` の場合：

```
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

展開したファイルを import したい場合は、次のようにできます。

```
import syssys.path.insert(0, "extracted")import requests
```

ただし、`requests` が依存するライブラリも別途必要です。展開だけでは依存関係は自動的に解決されません。

また、C拡張を含む wheel では、Python バージョン・OS・CPU アーキテクチャに適合する wheel を選ぶ必要があります。
