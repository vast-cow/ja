---
pubDatetime: 2026-04-14T17:22:31+09:00
title: "NVIDIA HPC SDK環境でNCCL Testsをビルドする方法"
description: "手順概要、各環境変数の意味、ビルドコマンドを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

NVIDIA HPC SDK環境で`nccl-tests`をビルドする場合は、CUDA、NCCL、MPIの各パスを正しく設定したうえで`make`を実行します。ここでは、必要な環境変数の設定とビルド手順を簡潔にまとめます。

## 手順概要

まず、NVIDIA HPC SDKのインストール先を基準に、CUDAとNCCLの場所を指定します。続いて、NCCLに同梱されているHPC-XのOpen MPIを`MPI_HOME`として設定します。

```bash
export CUDA_HOME="$NVHPC_ROOT/cuda"
export NCCL_HOME="$NVHPC_ROOT/comm_libs/nccl"
export MPI_HOME="$(readlink -f "$NCCL_HOME")/../hpcx/latest/ompi"
make -kj MPI=1 CUDA_HOME="$CUDA_HOME" NCCL_HOME="$NCCL_HOME" MPI_HOME="$MPI_HOME"
```

## 各環境変数の意味

### `CUDA_HOME`

`CUDA_HOME`には、NVIDIA HPC SDK内のCUDAディレクトリを指定します。

```bash
export CUDA_HOME="$NVHPC_ROOT/cuda"
```

これにより、ビルド時にCUDAのヘッダやライブラリを正しく参照できます。

### `NCCL_HOME`

`NCCL_HOME`には、HPC SDKに含まれるNCCLライブラリの場所を指定します。

```bash
export NCCL_HOME="$NVHPC_ROOT/comm_libs/nccl"
```

`nccl-tests`はこのパスを使ってNCCL本体を見つけます。

### `MPI_HOME`

`MPI_HOME`には、NCCLの周辺ライブラリとして配置されているHPC-XのOpen MPIを指定します。

```bash
export MPI_HOME="$(readlink -f "$NCCL_HOME")/../hpcx/latest/ompi"
```

`readlink -f`を使うことで、シンボリックリンクを解決した実体パスを基準に、適切なMPIディレクトリを参照できます。

## ビルドコマンド

環境変数を設定したあと、以下のコマンドで`nccl-tests`をビルドします。

```bash
make -kj MPI=1 CUDA_HOME="$CUDA_HOME" NCCL_HOME="$NCCL_HOME" MPI_HOME="$MPI_HOME"
```

### コマンドのポイント

* `MPI=1`
  MPI対応でビルドします。
* `CUDA_HOME=...`
  CUDAの参照先を明示します。
* `NCCL_HOME=...`
  NCCLの参照先を明示します。
* `MPI_HOME=...`
  使用するMPI実装の場所を指定します。
* `-kj`
  並列ビルドを有効にします。

## まとめ

NVIDIA HPC SDK環境で`nccl-tests`をビルドするには、`CUDA_HOME`、`NCCL_HOME`、`MPI_HOME`を正しく設定し、その値を`make`に渡すことが重要です。特に、MPIにはNCCL周辺に含まれるHPC-XのOpen MPIを指定する点がポイントです。これにより、HPC SDK環境に合わせた形で`nccl-tests`をスムーズにビルドできます。
