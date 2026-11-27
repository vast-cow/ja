---
title: "**Emacs に yaml-mode をインストールする手順**"
description: "署名エラーが発生した場合のGPG公開鍵の更新手順と、Emacsの設定ファイルの場所、NonGNU ELPAリポジトリの追加方法を解説。"
pubDatetime: 2026-11-27T17:12:00+09:00
---

## **1. GPG公開鍵の更新（署名エラーが発生する場合）**

以下のようなエラーが発生した場合に実施する。

```
Failed to verify signature archive-contents.sig:
No public key for 645357D2883A0966
```

Emacsで `M-:` を押し、以下のコードを入力して `RET` を押す。

```
(progn
  (require 'package)
  (let ((package-check-signature nil)
        (package-archives
         '(("gnu" . "https://elpa.gnu.org/packages/"))))
    (package-refresh-contents)
    (package-install 'gnu-elpa-keyring-update)))
```

GNU ELPAの公開鍵を更新する。署名検証はこの処理中のみ無効化され、処理終了後に元の設定に戻る。

なお、OSのパッケージ管理機能で公開鍵を更新できる場合は、そちらの方が安全である。

## **2. Emacsの設定ファイルの場所を確認**

Emacsで以下を実行する。

```
C-h v user-init-file RET
```

各操作の意味は以下のとおり。

| 操作               | 意味                              |
| ---------------- | ------------------------------- |
| `C-h`            | ヘルプ機能を呼び出す                      |
| `v`              | 変数の値と説明を表示する（describe-variable） |
| `user-init-file` | 起動時に読み込む設定ファイルを示す変数             |
| `RET`            | Enterキーで確定する                    |

今回の環境では、以下の結果が表示された。

```
Its value is "/home/naoto-aoki/.emacs"
```

したがって、設定ファイルは `~/.emacs` である。

Emacsの設定ファイルは環境によって異なる。

| 設定ファイルの候補                 |
| ------------------------- |
| `~/.emacs`                |
| `~/.emacs.el`             |
| `~/.emacs.d/init.el`      |
| `~/.config/emacs/init.el` |

`user-init-file` を確認することで、実際に使用されている設定ファイルを特定できる。

## **3. NonGNU ELPAリポジトリを追加**

設定ファイルを開く。

```
C-x C-f ~/.emacs RET
```

ファイルの末尾に移動する。

```
M->
```

以下のコードを追記する。

```
;; NonGNU ELPA
(require 'package)

(add-to-list 'package-archives
             '("nongnu" . "https://elpa.nongnu.org/nongnu/") t)
```

これにより、パッケージの取得先としてNonGNU ELPAが追加される。

## **4. 設定ファイルを保存**

```
C-x C-s
```

## **5. 設定を反映**

```
M-x eval-buffer RET
```

現在開いている設定ファイルのEmacs Lispコードを実行する。Emacsを再起動する必要はない。

## **6. パッケージ一覧を更新**

```
M-x package-refresh-contents RET
```

登録されているリポジトリから、インストール可能なパッケージの一覧を取得する。

## **7. yaml-modeをインストール**

```
M-x package-install RET yaml-mode RET
```

## **8. 動作確認**

YAMLファイルを開く。

```
C-x C-f test.yaml RET
```

例えば、以下の内容を入力する。

```
server:
  host: localhost
  port: 8080

database:
  name: example
  enabled: true
```

画面下部のモードラインに `YAML` と表示され、シンタックスハイライト（構文の色分け）が有効になれば成功。

以降は `.yaml` や `.yml` ファイルを開くと、自動的に `yaml-mode` が有効になる。

## **キー操作一覧**

| キー操作      | 機能               |
| --------- | ---------------- |
| `C-h v`   | 変数の値と説明を表示       |
| `C-h m`   | 現在のモードの説明を表示     |
| `C-x C-f` | ファイルを開く・新規作成     |
| `C-x C-s` | ファイルを保存          |
| `M-<`     | ファイルの先頭に移動       |
| `M->`     | ファイルの末尾に移動       |
| `M-:`     | Emacs Lisp式を実行   |
| `M-x`     | コマンドを実行          |
| `C-g`     | 操作をキャンセル         |
| `RET`     | Enterキー          |
| `C`       | Ctrlキー           |
| `M`       | Metaキー（通常はAltキー） |

## **参考リンク**

- [yaml-mode — GitHub](https://github.com/yoshiki/yaml-mode?utm_source=chatgpt.com)
- [NonGNU ELPA](https://elpa.nongnu.org/?utm_source=chatgpt.com)
- [GNU ELPA Keyring Update](https://elpa.gnu.org/packages/gnu-elpa-keyring-update.html?utm_source=chatgpt.com)
