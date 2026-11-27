---
title: "Emacsのsmerge-modeを単一キーで操作する"
description: "コンフリクト解消操作を単一キーに割り当て、モードラインにヘルプを表示するEmacs設定。"
pubDatetime: 2026-11-27T17:58:00+09:00
---

Emacsには、Gitのコンフリクトを解消するための `smerge-mode` が標準で用意されている。

コンフリクト解消時は操作がある程度限られるため、修飾キーを使わず、単一キーで操作できるようにした。

## 設定

```elisp
(with-eval-after-load 'smerge-mode
  ;; コンフリクト解消後に自動終了
  (setq smerge-auto-leave t)

  ;; コンフリクトマーカーの表示
  (set-face-attribute 'smerge-markers nil
                      :foreground 'unspecified
                      :background 'unspecified
                      :inherit 'default
                      :weight 'bold
                      :extend t)

  ;; キーバインド
  (define-key smerge-mode-map (kbd "n") #'smerge-next)
  (define-key smerge-mode-map (kbd "p") #'smerge-prev)
  (define-key smerge-mode-map (kbd "u") #'smerge-keep-upper)
  (define-key smerge-mode-map (kbd "l") #'smerge-keep-lower)
  (define-key smerge-mode-map (kbd "a") #'smerge-keep-all)
  (define-key smerge-mode-map (kbd "q")
              (lambda ()
                (interactive)
                (smerge-mode -1)))

  ;; モードラインにキーバインドのヘルプを表示
  (setf (alist-get 'smerge-mode minor-mode-alist)
        '(" SMerge [n]Next [p]Prev [u]Upper [l]Lower [a]All [q]Quit")))
```

## キーバインド

| キー | 操作 |
|---|---|
| `n` | 次のコンフリクトへ移動 |
| `p` | 前のコンフリクトへ移動 |
| `u` | 上側の変更を採用 |
| `l` | 下側の変更を採用 |
| `a` | 両方の変更を採用 |
| `q` | smerge-modeを終了 |

キーバインドを `smerge-mode-map` に定義しているため、モードが有効な間だけ適用される。

また、`minor-mode-alist` を変更し、モードラインにキーバインドのヘルプを表示するようにした。

```
SMerge [n]Next [p]Prev [u]Upper [l]Lower [a]All [q]Quit
```

## コンフリクトマーカーの表示調整

自分の環境では、ライトテーマを使用するとコンフリクトマーカーが黒背景に黒文字となり、視認性が悪かった。

そこで、`smerge-markers` の前景色と背景色の指定を解除し、`default` フェイスを継承するようにした。色はテーマに任せ、太字にすることでコンフリクトマーカーを強調している。

これにより、ライトテーマでもダークテーマでも使いやすい表示になる。

## コンフリクト解消後は自動終了

`smerge-auto-leave` が有効なので、smergeのコマンドで最後のコンフリクトを解消すると、モードが自動的に終了する。

キーバインドとヘルプ表示も自動的に解除され、通常の編集状態に戻る。途中で文字入力が必要になった場合は `q` で終了することもできる。

Emacsのマイナーモードの仕組みを利用するだけで、追加パッケージに依存せず、コンフリクト解消に特化した操作環境を作ることができた。
