---
title: "Codex CLIで承認確認をなくし、プロジェクト内の書き込みとネットワークアクセスを許可する方法"
description: "Codex CLIのconfig.tomlで、ワークスペース書き込み、承認なし、ネットワークアクセスを有効にする方法を解説。"
pubDatetime: 2026-10-20T13:15:00+09:00
---

OpenAIのCodex CLIを使っていると、ファイルの編集やコマンド実行のたびに承認を求められることがあります。

セキュリティ上は重要な仕組みですが、開発用のプロジェクトでCodexにある程度自律的に作業してもらいたい場合、承認確認が煩わしく感じることもあります。

そこで今回は、Codex CLIで以下の3つを設定する方法を紹介します。

- プロジェクト内のファイル書き込みを許可する
- コマンド実行時の承認確認を無効にする
- サンドボックス内からのネットワークアクセスを許可する

設定には`~/.codex/config.toml`を使用します。

## 1. Codex CLIの設定ファイルを編集する

Codex CLIの設定は、基本的に`~/.codex/config.toml`で管理します。

macOSまたはLinuxの場合、ターミナルで以下を実行します。

```bash
mkdir -p ~/.codex
nano ~/.codex/config.toml
```

すでに設定ファイルが存在する場合は、既存の設定を残しながら編集してください。

## 2. sandbox_modeとapproval_policyを設定する

設定ファイルに以下を記述します。

```toml
sandbox_mode = "workspace-write"
approval_policy = "never"

[sandbox_workspace_write]
network_access = true
```

これで、ワークスペース内の書き込みを許可し、承認確認を行わず、サンドボックス内でネットワークアクセスを利用する構成になります。

それぞれの設定を見ていきましょう。

### sandbox_mode = "workspace-write"

```toml
sandbox_mode = "workspace-write"
```

Codexがアクセスできるファイルシステムの範囲を指定する設定です。

`workspace-write`では、作業対象のワークスペース内への書き込みが許可されます。

なお、設定値は`project-write`ではなく、`workspace-write`です。

### approval_policy = "never"

```toml
approval_policy = "never"
```

Codexがコマンドを実行するときの承認ポリシーを指定します。

`never`にすると、ユーザーへの承認要求を行わずに作業を進めます。

ただし、承認確認を無効にすることと、すべての操作を許可することは別です。

サンドボックスによって禁止されている操作が自動的に許可されるわけではありません。

### network_access = true

```toml
[sandbox_workspace_write]
network_access = true
```

`workspace-write`サンドボックスでネットワークアクセスを許可します。

外部APIへのHTTPリクエストや、インターネットからの依存パッケージ取得など、ネットワークを必要とするコマンドを実行できるようにする設定です。

指定する値は文字列の`"enabled"`ではなく、真偽値の`true`です。

なお、ネットワークアクセスの可否には、実行環境や上位のセキュリティ制限も影響します。

## 3. コマンドラインから一時的に指定する方法

設定ファイルを変更せず、起動時のオプションで指定することもできます。

```bash
codex \
  --sandbox workspace-write \
  --ask-for-approval never \
  -c sandbox_workspace_write.network_access=true
```

この方法は、普段は承認確認を有効にしておき、特定の作業でだけ自律実行を許可したい場合に便利です。

設定ファイルに記述する方法と、起動時にオプションを指定する方法を用途に応じて使い分けるとよいでしょう。

## 4. 設定時の注意点

今回の設定では、Codexがユーザーの確認を待たずにファイルを変更したり、ネットワークにアクセスしたりできるようになります。

そのため、以下の点には注意が必要です。

- Gitで作業前の状態をコミットしておく
- `.env`などの機密情報を不用意に扱わせない
- 信頼できないリポジトリで安易に利用しない
- 本番環境の認証情報を使った状態で実行しない
- 必要に応じてサンドボックスや承認ポリシーを厳しくする

特に、`approval_policy = "never"`を指定すると、危険な操作に対して承認確認で気づく機会が減ります。

自動化の利便性と安全性のバランスを考えて設定しましょう。

## まとめ

Codex CLIでプロジェクト内の書き込み、承認確認なしの実行、ネットワークアクセスを有効にしたい場合は、以下の設定を使用します。

```toml
sandbox_mode = "workspace-write"
approval_policy = "never"

[sandbox_workspace_write]
network_access = true
```

設定のポイントは次の3つです。

1. `sandbox_mode`には`workspace-write`を指定する
2. `approval_policy`には`never`を指定する
3. ネットワークアクセスには`network_access = true`を指定する

これにより、Codex CLIをより自律的に動作させることができます。

ただし、承認確認を無効にしてもサンドボックスの制限は残るため、ワークスペース外への操作まで全面的に許可されるわけではありません。

ローカル開発や検証用の環境で、Codex CLIの作業効率を高めたい場合に活用してみてください。

参考資料： [OpenAI公式ドキュメント：Codex設定リファレンス](https://developers.openai.com/ja-JP/docs/config-file/config-reference)。`workspace-write`、`never`、`network_access`の設定値は公式仕様で確認しています。
