---
pubDatetime: 2026-07-16T20:06:01+09:00
title: "CUDA固有コードと通常のC++コードを分離する方法"
description: "ファイル構成、CUDA側の実装、C++側からCUDAシンボルを参照するを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

CUDAを使うプログラムでは、GPUカーネルやデバイスメモリを扱う部分を`.cu`ファイルに置き、通常のC++処理を`.cpp`ファイルに分けることができます。

この構成にすると、CUDA固有の処理とアプリケーション側の処理が混在しにくくなり、コードの役割が分かりやすくなります。

## ファイル構成

基本的には、次のように役割を分けます。

* `cuda.cu`

  * CUDAカーネル
  * `__constant__`メモリ
  * CUDA固有の関数
* `host.cpp`

  * `main`関数
  * 入出力
  * メモリ確保
  * カーネルの起動
  * 結果の確認

## CUDA側の実装

CUDA側では、constantメモリとカーネルを定義します。

```cpp
// cuda.cu

#include <cstdint>

// 長さ1のconstantメモリ
__constant__ std::uint32_t constant_value[1];

// constantメモリからグローバルメモリへコピーするカーネル
__global__ void copyConstantToGlobal(std::uint32_t* destination)
{
    if (blockIdx.x == 0 && threadIdx.x == 0) {
        destination[0] = constant_value[0];
    }
}
```

`constant_value`にはGPUから参照する値を保存します。

`copyConstantToGlobal`カーネルは、constantメモリに保存された値をグローバルメモリへコピーします。

## C++側からCUDAシンボルを参照する

通常のC++コンパイラは、`__constant__`や`__global__`といったCUDA固有のキーワードを処理できません。

そのため、C++側ではCUDAの修飾子を付けずに宣言します。

```cpp
// host.cpp

#include <cstdint>

extern std::uint32_t constant_value[1];

void copyConstantToGlobal(std::uint32_t* destination);
```

これらはC++側で実体を作る定義ではなく、別のファイルに存在するシンボルを参照するための宣言です。

## constantメモリへ値を書き込む

C++側からconstantメモリへ値を書き込む場合は、最初に`cudaGetSymbolAddress`でデバイス上のアドレスを取得します。

```cpp
void* constant_address = nullptr;

CUDA_CHECK(cudaGetSymbolAddress(
    &constant_address,
    constant_value
));
```

取得したアドレスに対して、`cudaMemcpy`で値を転送します。

```cpp
CUDA_CHECK(cudaMemcpy(
    constant_address,
    &input_value,
    sizeof(input_value),
    cudaMemcpyHostToDevice
));
```

これにより、ホスト側で生成した値を`constant_value[0]`へ保存できます。

## 出力用のグローバルメモリを確保する

カーネルの出力先として、GPU上にグローバルメモリを確保します。

```cpp
std::uint32_t* device_output = nullptr;

CUDA_CHECK(cudaMalloc(
    reinterpret_cast<void**>(&device_output),
    sizeof(*device_output)
));
```

ここでは、`std::uint32_t`を1個保存できる領域を確保しています。

## cudaLaunchKernelでカーネルを起動する

通常、`.cu`ファイル内では次の構文でカーネルを起動できます。

```cpp
copyConstantToGlobal<<<1, 1>>>(device_output);
```

しかし、この`<<<...>>>`構文は通常のC++コンパイラでは使用できません。

`.cpp`ファイルからカーネルを起動する場合は、`cudaLaunchKernel`を使用します。

### カーネル引数の準備

`cudaLaunchKernel`には、各カーネル引数を格納したホスト変数のアドレスを渡します。

今回のカーネル引数は次の1個です。

```cpp
std::uint32_t* destination
```

`device_output`が実際にカーネルへ渡すポインタ値なので、その変数のアドレスである`&device_output`を登録します。

```cpp
void* kernel_args[] = {
    &device_output
};
```

### カーネルの起動

```cpp
const dim3 grid_dim(1, 1, 1);
const dim3 block_dim(1, 1, 1);

CUDA_CHECK(cudaLaunchKernel(
    reinterpret_cast<const void*>(copyConstantToGlobal),
    grid_dim,
    block_dim,
    kernel_args,
    0,
    nullptr
));
```

各引数の意味は次のとおりです。

* `copyConstantToGlobal`

  * 起動するカーネル
* `grid_dim`

  * グリッドの大きさ
* `block_dim`

  * ブロックの大きさ
* `kernel_args`

  * カーネル引数
* `0`

  * 動的共有メモリのサイズ
* `nullptr`

  * デフォルトストリームを使用

今回の処理は1個の値をコピーするだけなので、1ブロック、1スレッドで起動します。

## カーネルの完了を待つ

カーネル起動後は、エラーの確認と処理完了の待機を行います。

```cpp
CUDA_CHECK(cudaGetLastError());
CUDA_CHECK(cudaDeviceSynchronize());
```

`cudaLaunchKernel`によるカーネル起動は非同期です。結果を読み出す前に`cudaDeviceSynchronize`を呼び、GPU処理が完了するまで待機します。

## 結果をホストへ戻す

GPU上の出力値をホスト側へコピーします。

```cpp
std::uint32_t output_value = 0;

CUDA_CHECK(cudaMemcpy(
    &output_value,
    device_output,
    sizeof(output_value),
    cudaMemcpyDeviceToHost
));
```

書き込んだ値と読み出した値を比較すれば、constantメモリとカーネルが正しく動作したか確認できます。

```cpp
const bool succeeded = input_value == output_value;

std::cout << "Result        : "
          << (succeeded ? "PASS" : "FAIL")
          << std::endl;
```

最後に、確保したGPUメモリを解放します。

```cpp
CUDA_CHECK(cudaFree(device_output));
```

## コンパイル方法

CUDAファイルは`nvcc`でコンパイルし、通常のC++ファイルは`g++`でコンパイルします。

```bash
nvcc -rdc=true -c cuda.cu
g++ -c host.cpp
nvcc cuda.o host.o -o constant_symbol_test
```

最後のリンク処理を`nvcc`で行うことが重要です。`nvcc`を使用することで、CUDAランタイムやCUDAデバイスコードに必要なリンク処理が実行されます。

`-rdc=true`は、CUDAコードを複数の翻訳単位に分けて扱うためのオプションです。

## この構成の目的

CUDAコードと通常のC++コードを分ける構成には、次の利点があります。

* GPU処理とホスト処理の役割を分離できる
* `.cpp`ファイルを通常のC++コンパイラでコンパイルできる
* CUDA固有の構文がアプリケーション全体に広がるのを防げる
* CUDA実装を独立したモジュールとして管理しやすくなる
* 既存のC++プロジェクトへCUDA処理を追加しやすくなる

重要な点は、CUDA固有の定義を`.cu`ファイルへ置き、C++側では必要なシンボルだけを通常のC++形式で宣言することです。

カーネル起動には`<<<...>>>`構文ではなく`cudaLaunchKernel`を使用し、最終的なリンクは`nvcc`で行います。
