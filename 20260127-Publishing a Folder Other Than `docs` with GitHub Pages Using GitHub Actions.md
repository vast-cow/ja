---
pubDatetime: 2026-01-27T21:43:50+09:00
title: "`docs` 以外のフォルダを GitHub Pages で公開する（GitHub Actions 方式）"
description: "GitHub Pages は /docs や特定ブランチから公開する構成がよく使われますが、frontend/ のような 任意のフォルダをそのまま公開したい場合は、GitHub Actions を使ったデプロイが安定です。リポジトリ側の設定が「Deploy from a branch」のままだと、A…"
---

GitHub Pages は `/docs` や特定ブランチから公開する構成がよく使われますが、`frontend/` のような **任意のフォルダをそのまま公開**したい場合は、**GitHub Actions を使ったデプロイ**が安定です。リポジトリ側の設定が「Deploy from a branch」のままだと、Actions の `deploy-pages` と噛み合わず問題になることがあります。

## GitHub Pages の公開ソースを GitHub Actions に切り替える

まず、リポジトリ設定を変更します。

1. リポジトリ → **Settings** → **Pages**
2. **Build and deployment** セクションへ
3. **Source** を **GitHub Actions** に設定

ここが **Deploy from a branch** のままだと、GitHub Actions の Pages 用ワークフロー（`actions/deploy-pages`）と競合・不整合が起きることがあります。

## GitHub Actions のワークフローを追加する

次に、以下のファイルを作成（または編集）します。

* `.github/workflows/pages.yml`

このワークフローは、リポジトリをチェックアウトした後に `frontend/` フォルダを Pages の成果物（artifact）としてアップロードし、GitHub Pages にデプロイします。

### ワークフローのポイント

* **実行トリガー**

  * `main` ブランチへの push で実行
  * `frontend/**` か `.github/workflows/pages.yml` に変更があるときだけ実行（無駄なデプロイを防ぐ）
  * `workflow_dispatch` により手動実行も可能

* **必要な権限**

  * Pages デプロイに必要な最小限の権限を付与：

    * `contents: read`
    * `pages: write`
    * `id-token: write`

* **デプロイ内容**

  * `frontend/` をそのまま公開物としてアップロード
  * `actions/deploy-pages` で GitHub Pages に反映

### `pages.yml` の例

```yaml
name: Deploy GitHub Pages (frontend)

on:
  push:
    branches: ["main"]
    # frontend/ に変更があったときだけデプロイ（スキップ要件）
    paths:
      - "frontend/**"
      - ".github/workflows/pages.yml"
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # frontend/ をそのまま Pages の公開物としてアップロード
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: frontend

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## 重要: 公開したいフォルダをそのままアップロードする

この手順の核心は、以下の指定です。

* `path: frontend`

これにより、`frontend/` が GitHub Pages の **サイトルート**として公開されます。`frontend/` 直下に `index.html` などの静的ファイルが揃っていれば、そのまま Web サイトとして配信されます。
