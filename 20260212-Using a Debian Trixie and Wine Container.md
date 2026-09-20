---
pubDatetime: 2026-02-12T20:47:18+09:00
title: "Debian Trixie + Wine コンテナの利用方法"
description: "この Dockerfile は、Debian Trixie をベースに Wine をインストールしたコンテナを作成します。主な目的は、Docker コンテナ内で Windows アプリケーションを実行することです。 Docker イメージのビルド まず、内容を Dockerfile という名前で保存…"
---

この Dockerfile は、Debian Trixie をベースに Wine をインストールしたコンテナを作成します。主な目的は、Docker コンテナ内で Windows アプリケーションを実行することです。



## Docker イメージのビルド

まず、内容を `Dockerfile` という名前で保存します。

次に、以下のコマンドでイメージをビルドします。

```bash
docker build -t debian-wine .
```

これにより、Wine Stable を含む `debian-wine` という名前の Docker イメージが作成されます。



## コンテナの起動方法

以下のコマンドで対話的にコンテナを起動します。

```bash
docker run -it --rm debian-wine
```

Dockerfile では次のように設定されています。

```dockerfile
CMD ["bash"]
```

そのため、起動すると Bash シェルが開きます。
作業ディレクトリは `/root` です。



## Windows アプリケーションの実行

コンテナ内で、次のように Windows 実行ファイルを起動できます。

```bash
wine your-program.exe
```

ホスト側のファイルを利用したい場合は、ボリュームマウントを使用します。

```bash
docker run -it --rm -v $(pwd):/work debian-wine
```

コンテナ内で以下を実行します。

```bash
cd /work
wine your-program.exe
```

これにより、ホスト上の Windows アプリケーションをコンテナ内の Wine で実行できます。



## 利用上のポイント

* 32bit（i386）アーキテクチャが有効化されているため、幅広い Windows アプリに対応可能
* WineHQ 公式の Stable 版を使用
* 非対話モードでビルドされるため自動化に適している
* 不要なキャッシュを削除しているため軽量

この構成により、Docker 上で安全かつクリーンな環境で Windows アプリケーションを実行できます。

```
FROM debian:trixie 

ENV DEBIAN_FRONTEND=noninteractive 

# Base packages 
RUN apt-get update && \ 
    apt-get install -y --no-install-recommends \ 
        ca-certificates \ 
        wget \ 
        gnupg \ 
        && rm -rf /var/lib/apt/lists/* 

# Enable i386 architecture (required for Wine) 
RUN dpkg --add-architecture i386 

# WineHQ keyring 
RUN mkdir -pm755 /etc/apt/keyrings && \ 
    wget -O /etc/apt/keyrings/winehq-archive.key \ 
      https://dl.winehq.org/wine-builds/winehq.key 

# WineHQ repository (Trixie) 
RUN wget -NP /etc/apt/sources.list.d/ \ 
      https://dl.winehq.org/wine-builds/debian/dists/trixie/winehq-trixie.sources 

# Install Wine 
RUN apt-get update && \ 
    apt-get install -y --install-recommends \ 
        winehq-stable && \ 
    rm -rf /var/lib/apt/lists/* 

WORKDIR /root 
CMD ["bash"] 
```
