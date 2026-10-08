---
title: "CodexをBubblewrapで隔離する：エージェントAI向けの現実的なサンドボックス構成"
description: "Bubblewrapでファイルシステムを読み取り専用にし、必要な場所だけ書き込み可能にする構成を紹介。CodexのExec ServerとClientを分離することで、プロジェクト内では自由に動作させつつ、ホームディレクトリやシステム設定への意図しない変更を防ぐ。"
pubDatetime: 2026-10-09T00:10:00+09:00
---

AIエージェントにシェルコマンドの実行権限を与えると、できることが一気に増える。

ファイルの編集、プログラムの実行、パッケージのインストール、ビルド、テスト。こうした作業を自律的に進められるのは便利だが、同時にホスト環境を意図せず破壊するリスクも生まれる。

とはいえ、エージェントAIを利用するたびに専用のVMやコンテナ環境を用意するのは面倒だ。普段使っているツールやライブラリにも、そのままアクセスさせたい。

そこで、Linuxの **Bubblewrap（bwrap）** を使い、Codexを隔離して動かす構成を考えた。

セキュリティ上の抜け道がまったくない構成ではない。それでも、自分が想定するエージェントAIの利用形態では、実用性と安全性のバランスが取れていると考えている。

## 何を防ぎたいのか

今回の目的は、悪意を持った攻撃者に対する完全な隔離ではない。

防ぎたいのは、エージェントAIが意図しないコマンドを実行したことで、作業対象以外のファイルを変更したり、普段使っている開発環境を壊したりすることだ。

例えば、エージェントが誤ってホームディレクトリ内のファイルを削除したり、システムの設定を書き換えたりするリスクを抑えたい。

一方で、プロジェクト内でのコード編集やビルド、テストなどは自由に実行させたい。

そのため、今回の方針はシンプルにした。

**ホストのファイルシステムは原則読み取り専用で公開し、必要な場所だけを書き込み可能にする。**

さらに、Codex本体とコマンド実行環境を分離するために、Codexの`exec-server`を利用する。

## 全体構成

今回の構成では、2つのBubblewrap環境を用意する。

```text
Host Linux
│
├── Bubblewrap ①
│   └── Codex Exec Server
│       ├── ホスト全体：原則 Read Only
│       ├── プロジェクト：Read / Write
│       ├── HOME：一時領域
│       └── WebSocket :8765
│
└── Bubblewrap ②
    └── Codex Client
        ├── ホスト全体：原則 Read Only
        ├── 作業ディレクトリ：output/ をマウント
        ├── ~/.codex：Read / Write
        └── Exec Server に接続
```

Codex ClientはAIとの対話や実行指示を担当し、Exec Serverはコマンドの実行を担当する。

両者はWebSocketで通信する。

重要なのは、**ClientとExec Serverでは、見えているファイルシステムと書き込み権限が異なる**ことだ。

Clientには`output/`を作業ディレクトリとして見せる。一方、Exec Serverには元のプロジェクト全体の読み書きを許可する。

このため、Clientから見えるファイルだけがエージェントの操作可能範囲になるわけではない。実際のコマンド実行にはExec Server側の権限が適用される。

これは今回、意図的に許容している仕様だ。

## 実装

Linux環境に`bwrap`とCodex CLIがインストールされていることを前提とする。

また、Codexの実行ファイルは`~/.codex/packages/standalone/current/bin/codex`に存在するものとする。`exec-server`の利用には、対応するバージョンのCodexが必要になる。

まず、プロジェクト内に出力用ディレクトリを作成する。

```bash
mkdir -p output
```

### ① Exec Serverの起動

1つ目のBubblewrapでは、コマンド実行専用のCodex Exec Serverを起動する。

```bash
bwrap \
  --die-with-parent \
  --unshare-pid \
  --unshare-ipc \
  --unshare-uts \
  \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  \
  --perms 0700 \
  --tmpfs "$HOME" \
  --ro-bind /etc/skel/.profile "$HOME/.profile" \
  --ro-bind /etc/skel/.bashrc "$HOME/.bashrc" \
  --ro-bind /etc/skel/.bash_logout "$HOME/.bash_logout" \
  \
  --dir "$HOME/.codex" \
  --ro-bind "$HOME/.codex/packages" "$HOME/.codex/packages" \
  \
  --dir "$HOME/.local" \
  --dir "$HOME/.local/bin" \
  --symlink \
    "$HOME/.codex/packages/standalone/current/bin/codex" \
    "$HOME/.local/bin/codex" \
  \
  --bind "$PWD" "$PWD" \
  \
  --tmpfs /tmp \
  --chmod 1777 /tmp \
  --tmpfs /var/tmp \
  --chmod 1777 /var/tmp \
  \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv PATH "$HOME/.local/bin:/usr/local/cuda/bin:/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  \
  --chdir "$PWD" \
  codex \
  --sandbox danger-full-access \
  --ask-for-approval never \
  exec-server \
  --listen ws://127.0.0.1:8765
```

この設定には、いくつかのポイントがある。

#### ホストのファイルシステムは読み取り専用

```bash
--ro-bind / /
```

ホストのルートファイルシステムを読み取り専用で公開する。

これによって、ホストにインストール済みのコマンドやライブラリをそのまま利用できる。

わざわざ隔離環境用のルートファイルシステムを構築する必要がないのは大きなメリットだ。

ただし、これはホストのファイルを秘匿する仕組みではない。OSのアクセス権限で読み取りが許されるファイルは基本的に見える。

#### ホームディレクトリは一時領域に置き換える

```bash
--perms 0700
--tmpfs "$HOME"
```

ホームディレクトリを`tmpfs`に置き換える。

これにより、元のホームディレクトリにある設定ファイルや認証情報などは、そのままでは見えなくなる。

シェルの初期化ファイルは`/etc/skel`から読み取り専用で配置する。

また、Codexの実行に必要なパッケージだけを公開する。

```bash
--dir "$HOME/.codex"
--ro-bind "$HOME/.codex/packages" "$HOME/.codex/packages"
```

Exec ServerにCodexの認証情報やユーザー設定全体を渡す必要はないため、このようにしている。

#### プロジェクトだけ書き込みを許可

```bash
--bind "$PWD" "$PWD"
```

これがExec Server側で最も重要な設定だ。

読み取り専用で公開したファイルシステムの上に、現在のプロジェクトディレクトリを読み書き可能な状態でマウントする。

これによって、エージェントはプロジェクト内のファイルを編集できる。

逆に、プロジェクト外の通常のホストファイルに対しては、ファイルシステム経由の書き込みが制限される。

もちろん、`/tmp`や一時HOMEなど、明示的に作成した書き込み可能領域は別だ。

#### 一時ディレクトリを分離する

```bash
--tmpfs /tmp
--chmod 1777 /tmp

--tmpfs /var/tmp
--chmod 1777 /var/tmp
```

ビルドやテストで使う一時ディレクトリも独立させている。

ホストの`/tmp`を共有しないため、通常の一時ファイル操作がホスト側に影響しない。

#### 環境変数を初期化する

```bash
--clearenv
```

ホストの環境変数を引き継がず、必要なものだけを明示的に設定する。

環境変数にはAPIキーやトークンなどが含まれることもあるため、不要な情報を渡さない設計にしている。

なお、プロキシや各種開発ツールが必要とする環境変数も消えるため、必要になったものは個別に追加する。

### ② Codex Clientの起動

続いて、別のターミナルからCodex Clientを起動する。

```bash
bwrap \
  --die-with-parent \
  --unshare-pid \
  --unshare-ipc \
  --unshare-uts \
  \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  \
  --perms 0700 \
  --tmpfs "$HOME" \
  --ro-bind /etc/skel/.profile "$HOME/.profile" \
  --ro-bind /etc/skel/.bashrc "$HOME/.bashrc" \
  --ro-bind /etc/skel/.bash_logout "$HOME/.bash_logout" \
  \
  --bind "$HOME/.codex" "$HOME/.codex" \
  \
  --dir "$HOME/.local" \
  --dir "$HOME/.local/bin" \
  --symlink \
    "$HOME/.codex/packages/standalone/current/bin/codex" \
    "$HOME/.local/bin/codex" \
  \
  --bind "$PWD/output" "$PWD" \
  \
  --tmpfs /tmp \
  --chmod 1777 /tmp \
  --tmpfs /var/tmp \
  --chmod 1777 /var/tmp \
  \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv PATH "$HOME/.local/bin:/usr/local/cuda/bin:/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  \
  --chdir "$PWD" \
  --setenv CODEX_EXEC_SERVER_URL ws://127.0.0.1:8765 \
  codex \
  --sandbox danger-full-access
```

Client側の構成はExec Serverとよく似ているが、重要な違いが2つある。

#### Codexの設定を共有する

```bash
--bind "$HOME/.codex" "$HOME/.codex"
```

Client側では、Codexの認証情報や設定が必要になるため、`~/.codex`を読み書き可能な状態でマウントする。

Exec Serverとは異なり、こちらには認証情報が見える。

つまり、認証情報の保護まで完全に実現する構成ではない。

#### 作業ディレクトリをoutputに差し替える

```bash
--bind "$PWD/output" "$PWD"
```

Clientにはプロジェクト全体ではなく、`output/`を現在の作業ディレクトリとして見せる。

例えば、ホスト上で次のような構成になっていたとする。

```text
project/
├── src/
├── README.md
├── package.json
└── output/
```

Client側では、ホストの`output/`が`project/`の位置にマウントされる。

一方、Exec Server側では元の`project/`全体が読み書き可能になっている。

そのため、このマウントによってExec Serverの操作範囲が`output/`だけに限定されるわけではない。

**Clientの作業空間を小さくすることと、コマンド実行権限を小さくすることは別の話だ。**

今回の構成では、後者についてはプロジェクト全体へのアクセスを許可する設計を採っている。

### ③ Exec Serverに接続する

最後に、次の環境変数を指定する。

```bash
--setenv CODEX_EXEC_SERVER_URL ws://127.0.0.1:8765
```

これによって、Codexは外部のExec Serverを実行環境として利用する。

Exec ServerとClientは同じホストのネットワーク名前空間を使用しているため、ループバックアドレスで通信できる。

ここで`--unshare-net`を指定していないのは意図的だ。

CodexがAPIへ通信するためのネットワーク接続を維持しつつ、Exec Serverとの通信も簡単に実現したかった。

## danger-full-accessでいいのか

今回の構成では、Codexに次の設定を使用している。

```bash
--sandbox danger-full-access
```

名前からして危険そうだが、この構成ではあえて採用している。

Codex側で追加のサンドボックス制御を重ねるのではなく、**OSレベルのBubblewrapによってファイルシステムへのアクセス権限を管理する**という考え方だ。

そのため、Codex側では強い制限をかけず、必要なコマンドを実行できるようにする。

ただし、`danger-full-access`自体に安全性があるわけではない。Bubblewrapで公開したファイル、ネットワーク、ソケットなどの権限は別途考える必要がある。

なお、元の例ではExec Server起動時にも`--sandbox`と`--ask-for-approval`を指定しているが、これらのCLIオプションだけでExec Serverの実効権限が制御されると考えるべきではない。

また、承認なしでの動作を明確にしたい場合は、Client側でも`--ask-for-approval never`を指定する。

この設計の中心となる保護機構は、あくまでBubblewrap側の設定だ。

## セキュリティ上の割り切り

この構成は万能ではない。

特に、以下の点は理解した上で利用する必要がある。

| リスク | 今回の対応 |
|---|---|
| プロジェクト外のファイルへの通常の書き込み | 原則として読み取り専用 |
| ホームディレクトリの誤操作 | tmpfsで分離 |
| 一時ファイルによるホスト環境への影響 | tmpfsで分離 |
| プロジェクト内のファイル破壊 | 許容する |
| ホスト上の読み取り可能な機密情報へのアクセス | 完全には防がない |
| ネットワーク経由の情報流出 | 防がない |
| 同一ホストからのExec Serverへの不正接続 | 認証なしのため防がない |

特に注意したいのは、`--ro-bind / /`ではホストの情報を完全には隠せないことだ。

ホストの他のディレクトリにある機密情報を読み取れる可能性があるし、ネットワーク通信も許可しているため、情報流出を防ぐ構成にはなっていない。

また、`/run`などに存在するUnix Domain Socketを経由したホストサービスへのアクセスにも注意が必要だ。読み取り専用のファイルシステムが、そのままサービス操作の禁止を意味するわけではない。

そして、Exec ServerのWebSocketは`127.0.0.1`で待ち受けているものの、今回の例では認証を設定していない。

外部ネットワークから直接公開しているわけではないが、同じホストの他のプロセスから接続できる可能性がある。

本格的なセキュリティ境界として利用するなら、WebSocketの認証、公開するディレクトリの削減、Unix Domain Socketの遮断、Network Namespaceの分離なども検討すべきだろう。

## それでもこの構成を採用する理由

ここまで読むと、セキュリティ上の対策が不十分に思えるかもしれない。

実際、敵対的なコードを安全に実行するための汎用サンドボックスとしては不十分だ。

しかし、自分が想定しているのは、信頼しているローカル環境上でCodexを実行し、エージェントによる予期しないファイル操作から普段の開発環境を守りたい、というユースケースだ。

プロジェクト内のファイルは、そもそもエージェントに編集させることを前提としている。

最悪の場合、プロジェクトが破壊される可能性はある。しかし、それはGitやバックアップなどで対処する範囲と考えている。

一方、普段使っているホームディレクトリやシステムの設定まで変更されるのは避けたい。

その目的に対して、Bubblewrapは非常に手軽だ。

コンテナイメージのメンテナンスも不要で、ホストのコマンドやライブラリを直接利用できる。DockerやVMと比較しても、自分の用途では運用上の負担が小さい。

もちろん、これは「十分なセキュリティ」を一般化するものではない。

機密情報を扱う環境や、第三者が提供した信頼できないコードを実行する用途であれば、もっと厳格な隔離を選ぶべきだと思う。

## まとめ

エージェントAIのセキュリティ対策を考えると、際限なく複雑な構成にできてしまう。

Network Namespaceの分離、seccomp、AppArmor、SELinux、コンテナ、VMなど、追加できる対策はいくらでもある。

しかし、セキュリティ対策は厳格であればよいというものでもない。

制限が強すぎてAIエージェントの利便性を損なったり、環境の維持管理に大きな手間がかかったりするのであれば、本来の目的から外れてしまう。

今回の構成では、BubblewrapとCodex Exec Serverを組み合わせることで、ホスト全体を自由に変更できる状態を避けながら、プロジェクト内ではエージェントを自由に動かすことを目指した。

防げない攻撃は残っているし、強固なセキュリティ境界として保証できるものではない。

それでも、自分の用途では、このくらいのシンプルな構成がちょうどよいと考えている。

**大切なのは、すべてのリスクをゼロにすることではなく、何を守る必要があり、何を許容できるのかを明確にした上で、必要な境界を設けることだ。**

---

### 参考資料

- [Bubblewrap — GitHub](https://github.com/containers/bubblewrap)
- [Bubblewrap — bwrap(1) Manual](https://manpages.debian.org/trixie/bubblewrap/bwrap.1.en.html)
- [Codex Exec Server — README](https://github.com/openai/codex/blob/main/codex-rs/exec-server/README.md)
- [Codex — Exec Serverを使ってTUIを起動するサンプル](https://github.com/openai/codex/blob/main/scripts/run_tui_with_exec_server.sh)

※ 本記事は構成例の紹介であり、実環境でのセキュリティ検証結果を示すものではありません。Codexの`exec-server`は実験的機能であり、バージョンによって設定や挙動が変わる可能性があります。
