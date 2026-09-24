---
pubDatetime: 2026-06-08T13:58:05+09:00
title: "Qiitaで注意書きを表示する方法"
description: "Qiita の note 記法、使い分け、実用例を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Qiita の注意書き系は、公式チートシート上では **`:::note` 系が基本**です。

## Qiita の `note` 記法

使える種類は主にこの3つです。Qiita公式の Markdown チートシートでは、`info` は省略可能、`warn` は警告、`alert` はより強い警告として説明されています。([Qiita][1])

```markdown
:::note
インフォメーション
info は省略可能です。
:::
```

:::note
インフォメーション
info は省略可能です。
:::

```markdown
:::note info
インフォメーション
:::
```

:::note info
インフォメーション
:::

```markdown
:::note warn
警告
○○に注意してください。
:::
```

:::note warn
警告
○○に注意してください。
:::

```markdown
:::note alert
より強い警告
○○しないでください。
:::
```

:::note alert
より強い警告
○○しないでください。
:::

## 使い分け

| 記法              | 用途                     |
| --------------- | ---------------------- |
| `:::note`       | 補足、メモ、参考情報             |
| `:::note info`  | `:::note` と同じ。明示したい場合  |
| `:::note warn`  | 注意、警告、破壊的でないリスク        |
| `:::note alert` | 強い警告、危険、破壊的操作、セキュリティ注意 |

## 実用例

```markdown
:::note warn
このコマンドはローカルの未コミット変更を上書きする可能性があります。
実行前に `git status` を確認してください。
:::
```

```markdown
:::note alert
本番データベースに対して直接実行しないでください。
データが復旧できなくなる可能性があります。
:::
```

## GitHub の Alert 記法とは別物

GitHub では次のような記法があります。

```markdown
> [!WARNING]
> This is a warning.
```

ただし、これは GitHub Flavored Markdown 側の Alert 記法で、Qiita の `:::note warn` とは別です。Qiita 向け記事なら、基本は `:::note warn` / `:::note alert` を使うのが安全です。

## 結論

Qiita で注意書きを入れるなら、通常はこれで十分です。

```markdown
:::note warn
注意事項を書きます。
:::
```

:::note warn
注意事項を書きます。
:::

より強く止めたい内容ならこちらです。

```markdown
:::note alert
絶対に本番環境で実行しないでください。
:::
```

:::note alert
絶対に本番環境で実行しないでください。
:::

[1]: https://qiita.com/Qiita/items/c686397e4a0f4f11683d?utm_source=chatgpt.com "Markdown記法 チートシート #Qiita"
