---
pubDatetime: 2026-04-13T18:51:43+09:00
title: "Slurmで確保したリソース上でインタラクティブシェルを開く"
description: "このスクリプトは、実行中のSLURMジョブにインタラクティブシェルをアタッチするための簡潔な方法を提供します。複数のコマンドラインツールを統合し、アクティブなジョブの特定、必要に応じたユーザー選択の促し、そしてsrunを用いた対象ジョブへのターミナルセッション接続を実現します。 概要 SLURMで管…"
---

このスクリプトは、実行中のSLURMジョブにインタラクティブシェルをアタッチするための簡潔な方法を提供します。複数のコマンドラインツールを統合し、アクティブなジョブの特定、必要に応じたユーザー選択の促し、そして`srun`を用いた対象ジョブへのターミナルセッション接続を実現します。

## 概要

SLURMで管理される高性能計算（HPC）環境では、実行中のジョブを確認・操作する必要が生じることがよくあります。このスクリプトは以下の処理を自動化します：

* 必要な依存関係の検証
* 現在のユーザーに属する実行中ジョブIDの取得
* 複数ジョブが存在する場合のインタラクティブ選択
* 選択したジョブ内でのインタラクティブシェルの起動

## 依存関係のチェック

スクリプトの冒頭で、必要なコマンドが利用可能かを確認します：

* `squeue`: SLURMジョブ情報の取得
* `jq`: JSON出力の解析
* `fzf`: インタラクティブな選択インターフェースの提供
* `srun`: SLURMジョブへのアタッチまたは起動

これらのツールのいずれかが欠けている場合、スクリプトはエラーを出して直ちに終了します。

## 実行中ジョブの取得

スクリプトは`squeue`のJSON出力を利用して、現在のユーザーの実行中ジョブを取得します：

```bash
squeue -u "$USER" --states Running --json 
```

この出力は`jq`で処理され、ジョブIDが抽出されます。`.jobs`または`.job_array`といった異なるJSON構造に対応し、空のエントリは除外されます。最終的なリストはソートされ、重複が排除されます。

## ジョブ選択の処理

検出されたジョブ数に応じて、スクリプトの動作は変わります：

* **ジョブなし**: 実行中ジョブが存在しない旨のメッセージを表示して終了
* **1件のみ**: そのジョブを自動的に選択
* **複数件**: `fzf`を使用してインタラクティブな選択メニューを表示

ユーザーが選択をキャンセルした場合、スクリプトは正常に終了します。

## ジョブへのアタッチ

ジョブIDが確定すると、以下のコマンドでインタラクティブシェルをアタッチします：

```bash
srun --jobid "$jobid" --overlap --pty /bin/bash --login -i 
```

### 主なオプションの説明

* `--jobid`: 対象ジョブの指定
* `--overlap`: 既存ジョブとリソースを共有して新しいステップを実行
* `--pty`: 疑似端末を割り当て
* `/bin/bash --login -i`: インタラクティブなログインシェルを起動

`exec`を使用することで、スクリプト自体が新しいシェルプロセスに置き換えられます。

## エラーハンドリングと堅牢性

このスクリプトは厳格なBash設定を使用しています：

```bash
set -euo pipefail 
```

これにより以下が保証されます：

* エラー発生時に即時終了（`-e`）
* 未定義変数の検出（`-u`）
* パイプライン内での適切なエラー伝播（`-o pipefail`）

これらの対策により、特に本番のHPC環境において信頼性が向上します。

## 結論

このスクリプトは、SLURMベースのシステムにおける一般的な運用ニーズである「実行中ジョブへの効率的なアタッチ」を簡潔に実現します。構造化データの解析、インタラクティブ選択、堅牢なエラーハンドリングを組み合わせることで、手動操作を削減し、複数ジョブ環境でのユーザーエラーを最小限に抑えます。

```bash
#!/usr/bin/env bash
set -euo pipefail

for cmd in squeue jq fzf srun column; do
  if ! command -v "$cmd" >/dev/null 2>&1; then
    echo "Error: '$cmd' not found." >&2
    exit 1
  fi
done

job_lines="$(
  squeue -u "$USER" --states Running --json \
    | jq -r '
        if .jobs then .jobs
        elif .job_array then .job_array
        else []
        end
        | map([
            (.job_id | tostring),
            (.name // "-"),
            (.partition // "-"),
            (.time_used // .run_time // "-"),
            (.nodes // .node_count // "-" | tostring),
            (.node_list // .nodelist // "-")
          ] | @tsv)
        | .[]
      '
)"

if [[ -z "$job_lines" ]]; then
  echo "No running jobs found." >&2
  exit 1
fi

job_count="$(printf '%s\n' "$job_lines" | grep -c '.')"

if (( job_count == 1 )); then
  jobid="$(printf '%s\n' "$job_lines" | cut -f1)"
else
  selected="$(
    {
      printf 'JOBID\tNAME\tPARTITION\tELAPSED\tNODES\tNODELIST\n'
      printf '%s\n' "$job_lines"
    } \
      | column -t -s $'\t' \
      | fzf \
          --prompt='Select job> ' \
          --height=50% \
          --reverse \
          --header-lines=1
  )"

  if [[ -z "${selected:-}" ]]; then
    echo "No job selected." >&2
    exit 1
  fi

  jobid="$(awk '{print $1}' <<< "$selected")"
fi

echo "Using jobid: $jobid" >&2
exec srun --jobid "$jobid" --overlap --pty /bin/bash --login -i
```
