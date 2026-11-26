---
title: "Codex CLIで推論過程を表示する方法"
description: "Codex CLIの設定ファイルや起動時オプションで推論表示を制御する方法を解説。"
pubDatetime: 2026-11-26T10:44:00+09:00
---

Codex CLIで推論過程（Reasoning）を表示するには、`~/.codex/config.toml` を編集します。

## **1. 設定ファイルで常時表示する**

```
vim ~/.codex/config.toml
```

以下の設定をトップレベルに追加してください。既に同じ設定項目がある場合は、その値を変更します。

```
hide_agent_reasoning = false
model_reasoning_summary = "detailed"
show_raw_agent_reasoning = true
```

設定を保存して、Codex CLIを再起動します。

| 設定項目                                   | 効果                   |
| -------------------------------------- | -------------------- |
| `hide_agent_reasoning = false`         | 推論イベントを非表示にしない       |
| `model_reasoning_summary = "detailed"` | 詳細な推論サマリーを要求する       |
| `show_raw_agent_reasoning = true`      | モデルが生の推論内容を返す場合に表示する |

これらはCodexの正式な設定項目です。

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+2



## **2. 起動時のオプションで一時的に有効化する**

設定ファイルを変更したくない場合は、以下のコマンドでも指定できます。

```
codex \
  -c 'hide_agent_reasoning=false' \
  -c 'model_reasoning_summary="detailed"' \
  -c 'show_raw_agent_reasoning=true'
```

## **3. 推論サマリーの詳細度**

`model_reasoning_summary` には以下の4種類を指定できます。

| 値          | 動作          |
| ---------- | ----------- |
| `auto`     | モデルに応じて自動選択 |
| `concise`  | 簡潔な推論サマリー   |
| `detailed` | 詳細な推論サマリー   |
| `none`     | 推論サマリーを無効化  |

また、推論能力自体を高めたい場合は、対応モデルで以下を追加します。

```
model_reasoning_effort = "high"
```

これは表示の詳細度とは独立した設定です。

## **4. 注意点**

**OpenAIのモデルでは、内部の思考過程（Chain of Thought）そのものは公開されていません。**

`show_raw_agent_reasoning = true` にしても、モデル側が生の推論内容を返さなければ表示されません。通常、閲覧できるのは推論サマリーです。

[image](https://www.google.com/s2/favicons?domain=https://developers.openai.com\&sz=32)

OpenAI API+1



また、CLI上で `Ctrl + T` を押すと、トランスクリプトを確認できます。

公式ドキュメント：

- [Configuration Reference](https://developers.openai.com/codex/config-reference?utm_source=chatgpt.com)
- [Advanced Configuration](https://developers.openai.com/codex/config-advanced?utm_source=chatgpt.com)
