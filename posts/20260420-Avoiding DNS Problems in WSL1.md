---
pubDatetime: 2026-04-20T17:01:48+09:00
title: "WSL1 の DNS 問題を回避する方法"
description: "何をしているのかについて、具体的な手順と注意点をまとめます。"
---

WSL1 を使っていると、名前解決が不安定になったり、`resolv.conf` が意図せず上書きされて DNS がうまく引けなくなったりすることがあります。
今回は、その問題をできるだけ安定して回避する方法をまとめます。

やることは大きく分けて 3 つです。

まず、**WSL が DNS 設定を自動生成しないようにする**ため、`/etc/wsl.conf` に `generateResolvConf = false` を設定します。
次に、**Windows 側で現在有効な DNS サーバーを取得**します。ここでは、既定の IPv4 / IPv6 ルートとインターフェースメトリックをもとに、実際に使われている可能性が高いアダプターを PowerShell で選びます。
最後に、**取得した DNS サーバー一覧を Linux の `nameserver` 形式に変換して `/etc/resolv.conf` に書き込む**ことで、安定した名前解決設定を作ります。途中で `tr -d '\r'` を使い、Windows 由来の CRLF も取り除きます。

## 手順

まずは Windows 側から対象のディストリビューションに入ります。

```bash
# Windows
wsl -d {distro}
```

WSL 側で `wsl.conf` を設定し、DNS 自動生成を無効化します。

```bash
# WSL
echo -e '[network]\ngenerateResolvConf = false' >> /etc/wsl.conf
exit
```

設定を反映するため、Windows 側で WSL を再起動します。

```bash
# Windows
wsl -t {distro}
wsl -d {distro}
```

再度 WSL に入ったら、既存の `resolv.conf` を削除し、Windows の PowerShell から取得した DNS サーバー情報を元に新しい `resolv.conf` を作成します。

```bash
# WSL
rm /etc/resolv.conf
/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -NoProfile -Command '$ifs=@(Get-NetRoute -DestinationPrefix "0.0.0.0/0","::/0" -ErrorAction SilentlyContinue | Sort-Object RouteMetric,InterfaceMetric | Select-Object -ExpandProperty InterfaceIndex -Unique); $dns=foreach($i in $ifs){ (Get-DnsClientServerAddress -InterfaceIndex $i).ServerAddresses }; $dns | Where-Object { $_ } | Select-Object -Unique | ForEach-Object { "nameserver $_" }' | tr -d '\r' > /etc/resolv.conf
```

## 何をしているのか

このコマンドでは、Windows 上で既定ルートに使われているネットワークインターフェースを優先度順に調べ、そのインターフェースに設定されている DNS サーバーを取り出しています。
その後、重複を除いたうえで `nameserver` 形式に変換し、Linux 側の `/etc/resolv.conf` に保存しています。

WSL1 では Windows のネットワーク設定に依存する場面が多いため、こうして **Windows 側の実際の DNS 設定をそのまま反映する**形にしておくと、手動で固定値を書くより環境変化に強くなります。

## 補足

この方法のポイントは、単純に `8.8.8.8` のようなパブリック DNS を直書きするのではなく、**Windows 側で現在有効な DNS をそのまま利用する**ことです。
社内ネットワークや VPN 環境でも動かしやすく、環境ごとの差異を吸収しやすいのが利点です。

WSL1 で DNS 周りが不安定な場合は、まずこの方法を試してみると改善することが多いと思います。
