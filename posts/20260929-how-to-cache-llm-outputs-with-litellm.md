---
title: "LiteLLMでLLMの出力をキャッシュする方法"
description: "Disk caching in LiteLLM stores LLM responses, reducing API costs and latency for repeated requests."
pubDatetime: 2026-09-29T23:14:00+09:00
---

LLMを使った開発では、同じプロンプトを何度も実行すると、そのたびに推論時間やAPIコストが発生します。

LiteLLMにはキャッシュ機能が用意されており、同一リクエストに対するLLMの出力を保存して再利用できます。ここでは、ディスクキャッシュを使う最小構成を紹介します。

まず、キャッシュとProxyに必要な依存関係をインストールします。

```bash
pip install 'litellm[caching,proxy]'
```

次に、LiteLLM Proxyの設定ファイルを作成します。

```yaml
model_list:
  - model_name: model
    litellm_params:
      model: openai/Qwen/Qwen3.6-35B-A3B # example
      api_key: ...
      api_base: ...

litellm_settings:
  cache: true
  cache_params:
    type: disk
    disk_cache_dir: ./Qwen_Qwen3.6-35B-A3B
```

`openai/Qwen/Qwen3.6-35B-A3B` というように `openai/` がありますが、これはOpenAI互換APIを使うことを示すために必要です。

ポイントは `litellm_settings` の設定です。

`cache: true` でキャッシュを有効化し、`cache_params.type` に `disk` を指定すると、LLMのレスポンスをローカルディスクに保存できます。

```yaml
cache_params:
  type: disk
  disk_cache_dir: ./Qwen_Qwen3.6-35B-A3B
```

この例では、キャッシュデータは `./Qwen_Qwen3.6-35B-A3B` ディレクトリに保存されます。

同じモデル・同じリクエストを再度送信した場合、キャッシュが利用できればLLMへの問い合わせを省略できるため、レスポンス時間の短縮やAPIコストの削減につながります。

特に、評価スクリプトやベンチマーク、開発中の繰り返しテストなど、同じ入力を何度も実行する用途では便利です。

LiteLLMを使っている場合は、まずディスクキャッシュから試してみると簡単です。
