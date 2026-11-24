---
title: "Codexで作業中にユーザーへ質問できるようにする設定「default_mode_request_user_input」"
description: "CodexのDefaultモードでユーザー入力要求を可能にする機能フラグ。config.tomlに2行追加するだけで、選択肢付きの質問を利用できる。"
pubDatetime: 2026-11-24T12:08:00+09:00
---

Codexに開発を任せていると、「ここは勝手に判断せず、自分に確認してほしかった」と感じることがあります。

例えば、複数の実装方法がある場合や、既存コードの大幅な変更が必要になった場合です。

そんなときに便利なのが、Codexの`default_mode_request_user_input`という機能フラグです。

`config.toml`にたった2行追加するだけで、通常のDefaultモードでも、Codexがユーザーに選択肢付きの質問を行えるようになります。

## default_mode_request_user_inputとは？

Codexには`request_user_input`という、ユーザーから追加情報を取得するためのツールがあります。

このツールを使うと、Codexは作業中にユーザーへ質問を提示し、回答を処理に反映できます。

もともとPlanモードで利用されていた機能ですが、Codex CLI v0.106.0でDefaultモードにも試験的に対応しました。

通常のチャットで質問するだけでなく、選択肢を提示するような対話型のワークフローを構築できるのが特徴です。

例えば、次のような場面で活用できます。

- 実装方法が複数あるときに、ユーザーに選択してもらう
- 仕様が曖昧な場合に、追加情報を確認する
- ファイルの構成変更前に、ユーザーの希望を確認する
- 設計や要件定義で、質問を段階的に行う

## 設定方法

設定は非常に簡単です。

### 1. config.tomlを開く

Codexの設定ファイルを編集します。

通常は以下の場所にあります。

```text
~/.codex/config.toml
```

ファイルが存在しない場合は作成してください。

### 2. featuresを追加する

設定ファイルに以下を記述します。

```toml
[features]
default_mode_request_user_input = true
```

すでに`[features]`セクションが存在する場合は、その中に`default_mode_request_user_input = true`だけを追加します。

これで設定は完了です。Codexを再起動して反映させましょう。

### 3. 設定が有効になっているか確認する

ターミナルで以下を実行します。

```bash
codex features list
```

`default_mode_request_user_input`が有効になっていれば設定できています。

設定ファイルを編集せずに、一時的に有効化することも可能です。

```bash
codex --enable default_mode_request_user_input
```

## 実際に使ってみる

例えば、Codexに次のように指示します。

> READMEを改善してください。
>
> ただし、作業を始める前に`request_user_input`を使用して、対象読者が「初心者」か「経験者」かを質問してください。
>
> 回答に合わせてREADMEの内容を調整してください。

対応する環境では、Codexが選択肢付きの質問を表示できます。

ユーザーが回答すると、その内容を参考に作業を進められます。

重要なのは、質問するタイミングをCodexに任せるだけでなく、**プロンプト側で明示的に質問を指示できる**ことです。

これによって、Codexとのやり取りを単なる作業依頼から、対話を伴う開発ワークフローへと拡張できます。

## 注意点：無期限の待機ができるわけではない

ここは注意が必要です。

`default_mode_request_user_input = true`は、Defaultモードで質問ツールを使えるようにする設定であり、ユーザーが回答するまで必ず停止し続ける設定ではありません。

Defaultモードでは、一定時間内に回答しなかった場合、質問が自動的に解決される動作が報告されています。

そのため、次のような用途では注意が必要です。

- ユーザーの承認があるまで本番環境を変更しない
- 数時間後の回答まで作業を停止する
- 明示的な許可が得られるまで処理を再開しない

こうした厳密な承認制御では、質問機能だけに依存せず、別途承認フローや待機状態を管理する仕組みを用意する必要があります。

また、`request_user_input`とCodexのコマンド実行承認ポリシーは別の仕組みです。

## MCPは必要？

今回紹介した機能を使うだけなら、MCPサーバーを追加する必要はありません。

`request_user_input`はCodex側の機能です。

一方、独自のMCPツールがユーザー入力を要求する場合は、MCPのElicitationという別の仕組みを使用します。

つまり、Codex自身が仕様を確認する用途なら`request_user_input`、独自ツールからユーザー入力を要求する用途ならMCP Elicitation、と使い分けられます。

## まとめ

Codexに次の設定を追加すると、通常のDefaultモードでもユーザー入力要求機能を利用できるようになります。

```toml
[features]
default_mode_request_user_input = true
```

特に、要件定義、設計方針の選択、既存コードの改修など、ユーザーの判断が必要になる開発作業と相性のよい機能です。

ただし、現時点では開発中の機能であり、Codexのバージョンや使用環境によって動作が異なる場合があります。

個人的には、Codexを単にコードを書くAIとして使うのではなく、**必要な場面で質問しながら開発を進めるエージェント**として活用するための、便利な設定だと考えています。

## 参考資料

- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference)
- [Codexの機能フラグ実装（GitHub）](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)
- [Defaultモードへのrequest_user_input対応](https://github.com/openai/codex/issues/10384)
- [Defaultモードでの無期限待機に関する議論](https://github.com/openai/codex/issues/37472)

*2026年11月時点の情報を基に作成。*
