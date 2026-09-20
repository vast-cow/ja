---
pubDatetime: 2026-06-26T12:09:52+09:00
title: "MSVC PowerShell の起動"
description: "目的、動作の仕組み、コマンドライン オプションを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

この例では、Microsoft Visual C++（MSVC）開発環境用にあらかじめ構成された PowerShell セッションを起動する方法を示します。

このスクリプトは、Visual Studio Developer PowerShell を手動で開く代わりに、適切な Visual Studio または Build Tools のインストールを自動的に検出し、MSVC が有効化された PowerShell セッションを起動します。

## 目的

このスクリプトは、次のような場合に役立ちます。

* スクリプトから MSVC 開発環境を起動したい。
* Visual Studio IDE を起動せずに Visual Studio Build Tools を利用したい。
* 特定のターゲット アーキテクチャまたはホスト アーキテクチャを選択したい。
* インストールされている最新の Visual Studio または Build Tools を自動的に使用したい。

## 動作の仕組み

PowerShell スクリプトは、以下の手順を実行します。

1. コマンドライン オプションを解析します。
2. `vswhere.exe` を使用して Visual Studio のインストールを検索します。
3. Visual Studio Developer Shell モジュールを読み込みます。
4. `Enter-VsDevShell` を呼び出して Developer Shell を起動します。

適切なインストールが見つからない場合は、必要なコンポーネントを説明するエラーメッセージを表示します。

## コマンドライン オプション

このスクリプトは、以下のオプションをサポートしています。

| オプション          | 説明                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--arch`       | ターゲット アーキテクチャ（`x86`、`amd64`、`arm`、または `arm64`）。                                                                             |
| `--host-arch`  | ホスト アーキテクチャ。                                                                                                                |
| `--product`    | 検索対象の Visual Studio 製品。既定値は `Microsoft.VisualStudio.Product.BuildTools` です。すべてのインストール済み Visual Studio 製品を検索するには `*` を指定します。 |
| `--prerelease` | プレリリース版の Visual Studio インストールも検索対象に含めます。                                                                                    |
| `--help`       | 使用方法を表示します。                                                                                                                 |

## スクリプトの起動

次のシェル スクリプトは、PowerShell スクリプトを起動します。

```powershell
powershell.exe -NoLogo -NoExit -ExecutionPolicy Bypass -File ./msvc-devshell.ps1
```

このスクリプトを実行すると、MSVC 開発環境があらかじめ構成された PowerShell セッションが起動します。セッションは `-NoExit` オプションを指定して開始されるため、初期化完了後も PowerShell ウィンドウは閉じず、そのまま `cl`、`link`、`nmake` などのビルド ツールを実行できます。

```powershell
$ErrorActionPreference = "Stop"

$Arch = "amd64"
$HostArch = "amd64"
$Product = "Microsoft.VisualStudio.Product.BuildTools"
$Prerelease = $false

function Show-Usage {
@"
Usage:
  msvc-pwsh.ps1 [options]

Options:
  --arch ARCH          Target architecture: x86, amd64, arm, arm64
  --host-arch ARCH     Host architecture: x86, amd64, arm, arm64
  --product PRODUCT    Visual Studio product id.
                       Default: Microsoft.VisualStudio.Product.BuildTools
                       Use "*" to search all Visual Studio products.
  --prerelease         Include prerelease Visual Studio instances.
  -h, --help           Show this help.
"@
}

$argv = @($args)

$i = 0
while ($i -lt $argv.Count) {
  switch ($argv[$i]) {
    { $_ -in @("-arch", "--arch") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $Arch = $argv[$i + 1]
      $i += 2
      continue
    }

    { $_ -in @("-host_arch", "--host-arch") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $HostArch = $argv[$i + 1]
      $i += 2
      continue
    }

    { $_ -in @("-product", "--product") } {
      if ($i + 1 -ge $argv.Count) {
        [Console]::Error.WriteLine("missing value for $($argv[$i])")
        exit 2
      }
      $Product = $argv[$i + 1]
      $i += 2
      continue
    }

    "--prerelease" {
      $Prerelease = $true
      $i += 1
      continue
    }

    { $_ -in @("-h", "--help") } {
      Show-Usage
      exit 0
    }

    default {
      [Console]::Error.WriteLine("unknown argument: $($argv[$i])")
      exit 2
    }
  }
}

$validArch = @("x86", "amd64", "arm", "arm64")

if ($Arch -notin $validArch) {
  [Console]::Error.WriteLine("invalid --arch: $Arch")
  exit 2
}

if ($HostArch -notin $validArch) {
  [Console]::Error.WriteLine("invalid --host-arch: $HostArch")
  exit 2
}

$programFilesX86 = ${Env:ProgramFiles(x86)}
if (-not $programFilesX86) {
  $programFilesX86 = "C:\Program Files (x86)"
}

$vswhere = Join-Path $programFilesX86 "Microsoft Visual Studio\Installer\vswhere.exe"

if (-not (Test-Path -LiteralPath $vswhere -PathType Leaf)) {
  [Console]::Error.WriteLine("vswhere.exe not found: $vswhere")
  exit 1
}

$vswhereArgs = @(
  "-latest",
  "-products", $Product,
  "-requires", "Microsoft.VisualStudio.Component.VC.Tools.x86.x64",
  "-property", "installationPath"
)

if ($Prerelease) {
  $vswhereArgs += "-prerelease"
}

$installationPath = & $vswhere @vswhereArgs |
  ForEach-Object { $_ -replace "`r$", "" } |
  Select-Object -First 1

if (-not $installationPath) {
  [Console]::Error.WriteLine(@"
Visual Studio / Build Tools with MSVC was not found.

Try:
  .\msvc-pwsh.ps1 --product "*"

Or install:
  - Visual Studio Build Tools
  - MSVC C++ build tools
  - Windows SDK
"@)
  exit 1
}

$devshellDll = Join-Path $installationPath "Common7\Tools\Microsoft.VisualStudio.DevShell.dll"

if (-not (Test-Path -LiteralPath $devshellDll -PathType Leaf)) {
  [Console]::Error.WriteLine("DevShell module not found: $devshellDll")
  exit 1
}

Set-Location $Env:USERPROFILE

Import-Module $devshellDll

Enter-VsDevShell `
  -VsInstallPath $installationPath `
  -SkipAutomaticLocation `
  -Arch $Arch `
  -HostArch $HostArch `
  -DevCmdArguments "-no_logo"
```
