---
title: "Codex で MCP や画像入力がエラーになるときに試したい LiteLLM の `use_chat_completions_api`"
description: "LiteLLM の `use_chat_completions_api: true` 設定で、Codex CLI の Responses API リクエストを Chat Completions API に変換し、ツール呼び出しや画像入力のエラーを回避する方法を解説。"
pubDatetime: 2026-11-25T15:26:00+09:00
---

Codex などを OpenAI 互換のモデルサーバーにつなぐと、テキストだけのやり取りはできるのに、MCP ツールを呼び出したり、スクリーンショットを添付したりするとエラーになることがあります。Codex CLI はモデルとの通信に Responses API を使いますが、接続先によっては、ツール呼び出しや画像入力を Chat Completions API でしか受け付けない場合があります。[LiteLLM の Codex 接続ガイド](https://docs.litellm.ai/docs/proxy/client_setup/codex_cli) · [OpenAI の画像入力ガイド](https://developers.openai.com/api/docs/guides/images-vision)

そんなときは、LiteLLM で `use_chat_completions_api: true` を試してみましょう。Codex は `/v1/responses` でリクエストを送り、LiteLLM が接続先の `/v1/chat/completions` に合わせて変換します。接続先が Responses API に対応しておらず、テキストだけでも 404 になる場合にも使える設定です。[LiteLLM の Responses API ガイド](https://docs.litellm.ai/docs/response_api#opt-in-bridge-for-openai-models-with-custom-api_base)

## エラーが出る入力を一つずつ試す

まずテキストだけを送り、次に画像を１枚添付した場合と、MCP ツールを１つ呼び出した場合をそれぞれ試します。LiteLLM と接続先のログを見て、モデル API がリクエストを拒否しているなら、同じモデルの `/v1/chat/completions` が該当する画像入力や関数呼び出しを受け付けるか確認します。そこで通るなら、変換を試す価値があります。MCP サーバーへの接続や認証の段階で失敗している場合は、サーバーの設定を確認しましょう。[LiteLLM の画像入力ガイド](https://docs.litellm.ai/docs/completion/vision) · [Codex の MCP 設定ガイド](https://developers.openai.com/codex/mcp)

## LiteLLM Proxy で対象モデルに設定する

`config.yaml` の対象モデルの `litellm_params` に `use_chat_completions_api: true` を追加します。LiteLLM はクライアントから `/v1/responses` で受け取り、接続先には `/v1/chat/completions` でリクエストを送り、結果を Responses API 形式に変換して返します。[LiteLLM 公式ドキュメント](https://docs.litellm.ai/docs/response_api)

```yaml
model_list:
  - model_name: local-chat
    litellm_params:
      model: openai/my-served-model
      api_base: http://127.0.0.1:8080/v1
      api_key: os.environ/UPSTREAM_API_KEY
      use_chat_completions_api: true

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

`my-served-model` は接続先が公開しているモデル ID に置き換えます。クライアントから指定する名前は `local-chat` です。この例では、接続先のキーを `UPSTREAM_API_KEY` に、Proxy のマスターキーを `LITELLM_MASTER_KEY` に設定してから起動します。[LiteLLM の YAML 設定ガイド](https://docs.litellm.ai/docs/proxy/configs)

```bash
litellm --config config.yaml --port 4000
```

Codex CLI から使う場合は、`~/.codex/config.toml` のモデル名に `local-chat` を指定します。LiteLLM 用のプロバイダー設定では `base_url = "http://127.0.0.1:4000/v1"`、`env_key = "LITELLM_API_KEY"`、`wire_api = "responses"` を使います。[LiteLLM の Codex 接続ガイド](https://docs.litellm.ai/docs/proxy/client_setup/codex_cli)

このローカル検証では、別のターミナルで `LITELLM_MASTER_KEY` と同じ値を `LITELLM_API_KEY` に設定して呼び出します。

```bash
curl -sS http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"model":"local-chat","input":"日本語で短く挨拶してください。"}'
```

レスポンスの `output` にテキストが入り、接続先のログに `POST /v1/chat/completions` が記録されていれば、変換が機能しています。

## Python SDK では `True` を渡す

Proxy を介さず LiteLLM の Python SDK から接続する場合は、`litellm.responses()` に同じ設定をキーワード引数として渡せます。[LiteLLM 公式ドキュメント](https://docs.litellm.ai/docs/response_api)

```python
import os

import litellm

response = litellm.responses(
    model="openai/my-served-model",
    api_base="http://127.0.0.1:8080/v1",
    api_key=os.environ["UPSTREAM_API_KEY"],
    input="日本語で短く挨拶してください。",
    use_chat_completions_api=True,
)
print(response)
```

## 切り替えたら、エラーが出た操作をもう一度試す

短いテキストが返ったら、Codex で元の画像を添付し直し、画像の内容に沿った返答が返るか確認します。MCP は、ツールが実際に呼ばれ、その結果を使って応答できるところまで試しましょう。画像入力には接続先のモデルと API の画像対応が必要です。MCP ツールの動作も、Codex 側の接続設定や、モデルに渡されるツール形式への対応に左右されます。[LiteLLM の画像入力ガイド](https://docs.litellm.ai/docs/completion/vision) · [LiteLLM のツール変換に関する報告](https://github.com/BerriAI/litellm/issues/35878)
