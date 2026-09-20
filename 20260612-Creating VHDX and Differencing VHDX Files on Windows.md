---
pubDatetime: 2026-06-12T22:05:52+09:00
title: "WindowsでVHDXを作成する方法と差分VHDXを作成する方法"
description: "前提 WindowsでVHDXを作成する方法は主に次の3つです。 | 方法 | 用途 | | ------------------------------ | ------------------------------ | | ディスクの管理 GUI | 手作業で簡単に作る | | PowerSh…"
---

## 前提

WindowsでVHDXを作成する方法は主に次の3つです。

| 方法                             | 用途                             |
| ------------------------------ | ------------------------------ |
| **ディスクの管理 GUI**                | 手作業で簡単に作る                      |
| **PowerShell / Hyper-V モジュール** | 自動化、差分VHDX作成に向く                |
| **DiskPart**                   | Windows標準CLIで作る、Hyper-Vモジュール不要 |

差分VHDXを作る場合は、通常 **PowerShell の `New-VHD -Differencing`** を使うのが最も簡単です。Microsoft公式ドキュメントにも、`New-VHD -ParentPath c:\Base.vhdx -Path c:\Diff.vhdx -Differencing` という差分VHDX作成例があります。([Microsoft Learn][1])

---

# 1. 通常のVHDXを作成する方法

## 方法A: GUI「ディスクの管理」で作成

1. `Win + X`
2. **ディスクの管理** を開く
3. メニューから
   **操作** → **VHD の作成**
4. 以下を指定する

   * 場所: `C:\VHD\Base.vhdx`
   * サイズ: 例 `64 GB`
   * 形式: **VHDX**
   * 種類:

     * **固定サイズ**: 最初から全容量を確保。性能・安定性重視
     * **可変容量**: 使用量に応じてファイルが増える。容量節約向き
5. 作成後、ディスク一覧に追加される
6. 右クリックして **ディスクの初期化**
7. GPT または MBR を選択

   * 通常は **GPT**
8. 未割り当て領域を右クリック
9. **新しいシンプル ボリューム**
10. NTFS などでフォーマット

---

## 方法B: PowerShellでVHDXを作成

管理者権限のPowerShellで実行します。

### 可変容量VHDXを作成

```powershell
New-VHD -Path "C:\VHD\Base.vhdx" -SizeBytes 64GB -Dynamic
```

### 固定サイズVHDXを作成

```powershell
New-VHD -Path "C:\VHD\Base.vhdx" -SizeBytes 64GB -Fixed
```

`New-VHD` は仮想ハードディスクを作成するHyper-V PowerShellコマンドレットです。MicrosoftのHyper-Vモジュールにも `New-VHD` が含まれています。([Microsoft Learn][2])

作成したVHDXをマウントして初期化・フォーマットする例です。

```powershell
Mount-VHD -Path "C:\VHD\Base.vhdx"

$disk = Get-Disk | Where-Object PartitionStyle -eq "RAW" | Sort-Object Number -Descending | Select-Object -First 1

Initialize-Disk -Number $disk.Number -PartitionStyle GPT

New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter V |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "BaseVHDX" -Confirm:$false
```

---

## 方法C: DiskPartでVHDXを作成

Hyper-V PowerShellモジュールを使わず、Windows標準の `diskpart` で作成できます。MicrosoftのDiskPartリファレンスでは、`create vdisk` は仮想ハードディスクを作成し、作成直後は未初期化ディスクと同じ状態になると説明されています。([Microsoft Learn][3])

管理者権限のコマンドプロンプトで実行します。

```cmd
diskpart
```

DiskPart内で以下を実行します。

```cmd
create vdisk file="C:\VHD\Base.vhdx" maximum=65536 type=expandable
select vdisk file="C:\VHD\Base.vhdx"
attach vdisk
create partition primary
format fs=ntfs quick label="BaseVHDX"
assign letter=V
exit
```

ポイント:

```cmd
maximum=65536
```

は **MB単位** なので、65536 MB = 64 GB です。

```cmd
type=expandable
```

は可変容量です。固定サイズにしたい場合は次のようにします。

```cmd
create vdisk file="C:\VHD\Base.vhdx" maximum=65536 type=fixed
```

`attach vdisk` はVHD/VHDXをマウントし、ローカルディスクとして表示するコマンドです。([Microsoft Learn][4])

---

# 2. 別のVHDXをベースに差分VHDXを作成する方法

## 差分VHDXの考え方

差分VHDXは、親VHDXを読み取り元として使い、変更分だけを子VHDXに保存する形式です。

```text
Base.vhdx        ← 親ディスク。基本イメージ
  └─ Diff01.vhdx ← 子ディスク。変更分だけ保存
```

たとえば、以下のような用途に向いています。

* 検証環境をすばやく複製する
* クリーンなOSイメージを親として保持する
* 複数のテスト環境を小さい容量で作る
* 壊しても差分VHDXだけ削除すれば戻せる

---

## 重要な注意点

差分VHDXを作る前に、親VHDXについて次を守る必要があります。

| 注意点                | 理由                   |
| ------------------ | -------------------- |
| 親VHDXは変更しない        | 子VHDXとの整合性が壊れる可能性がある |
| 親VHDXの場所を不用意に変えない  | 子VHDXが親を見つけられなくなる    |
| 親VHDXを削除しない        | 子VHDXが使用不能になる        |
| 親VHDXは読み取り専用にすると安全 | 誤更新を防げる              |
| 差分チェーンを深くしすぎない     | 性能低下・管理複雑化の原因になる     |

親VHDXを保護する例:

```powershell
Set-ItemProperty -Path "C:\VHD\Base.vhdx" -Name IsReadOnly -Value $true
```

解除する場合:

```powershell
Set-ItemProperty -Path "C:\VHD\Base.vhdx" -Name IsReadOnly -Value $false
```

---

# 3. PowerShellで差分VHDXを作成する

## 基本コマンド

```powershell
New-VHD `
  -ParentPath "C:\VHD\Base.vhdx" `
  -Path "C:\VHD\Diff01.vhdx" `
  -Differencing
```

Microsoft公式ドキュメントでも、`-ParentPath` に親VHDX、`-Path` に作成する差分VHDX、`-Differencing` を指定する例が示されています。([Microsoft Learn][1])

---

## 作成後にマウントする

```powershell
Mount-VHD -Path "C:\VHD\Diff01.vhdx"
```

マウントすると、Windowsからは通常のディスクのように見えます。

ただし、差分VHDXは親VHDXの内容をベースにしているため、通常は親側にすでにパーティションやファイルシステムが存在します。その場合、初期化やフォーマットは不要です。

---

## 差分VHDXをアンマウントする

```powershell
Dismount-VHD -Path "C:\VHD\Diff01.vhdx"
```

---

# 4. 差分VHDXを使った実用例

## 例: ベースOSイメージからテスト環境を作る

```powershell
# 親VHDX
$parent = "D:\VHD\BaseWindows.vhdx"

# 差分VHDX
$child = "D:\VHD\Test01.vhdx"

# 差分ディスク作成
New-VHD -ParentPath $parent -Path $child -Differencing

# マウント
Mount-VHD -Path $child
```

複数作る場合:

```powershell
New-VHD -ParentPath "D:\VHD\BaseWindows.vhdx" -Path "D:\VHD\Test01.vhdx" -Differencing
New-VHD -ParentPath "D:\VHD\BaseWindows.vhdx" -Path "D:\VHD\Test02.vhdx" -Differencing
New-VHD -ParentPath "D:\VHD\BaseWindows.vhdx" -Path "D:\VHD\Test03.vhdx" -Differencing
```

構成はこうなります。

```text
BaseWindows.vhdx
 ├─ Test01.vhdx
 ├─ Test02.vhdx
 └─ Test03.vhdx
```

各差分VHDXは独立した変更分を持ちます。

---

# 5. 差分VHDXの親パスを確認・修正する

親VHDXの場所を移動した場合、子VHDXが親を見つけられなくなることがあります。

その場合は `Set-VHD -ParentPath` で親パスを設定できます。Microsoftの `Set-VHD` ドキュメントでは、差分VHDの親ディスクのパスを指定する `-ParentPath` パラメータが説明されています。([Microsoft Learn][5])

```powershell
Set-VHD `
  -Path "C:\VHD\Diff01.vhdx" `
  -ParentPath "D:\VHD\Base.vhdx"
```

---

# 6. 差分VHDXを親VHDXに統合する

差分VHDXの内容を親または別の差分ディスクに統合したい場合は `Merge-VHD` を使います。Microsoftの説明では、`Merge-VHD` は差分VHDチェーン内の仮想ハードディスクをマージするコマンドレットです。([Microsoft Learn][6])

例:

```powershell
Merge-VHD -Path "C:\VHD\Diff01.vhdx" -DestinationPath "C:\VHD\Base.vhdx"
```

ただし、親VHDXに直接統合するとベースイメージが変わります。クリーンなベースを維持したい場合は、統合前にバックアップを取るか、別ファイルへ変換する方が安全です。

---

# 7. 最小構成のコマンドまとめ

## 通常のVHDX作成

```powershell
New-VHD -Path "C:\VHD\Base.vhdx" -SizeBytes 64GB -Dynamic
```

## マウント

```powershell
Mount-VHD -Path "C:\VHD\Base.vhdx"
```

## 差分VHDX作成

```powershell
New-VHD -ParentPath "C:\VHD\Base.vhdx" -Path "C:\VHD\Diff01.vhdx" -Differencing
```

## 差分VHDXマウント

```powershell
Mount-VHD -Path "C:\VHD\Diff01.vhdx"
```

## アンマウント

```powershell
Dismount-VHD -Path "C:\VHD\Diff01.vhdx"
```

---

## 推奨運用

検証用途なら、次の構成が扱いやすいです。

```text
D:\VHD\
 ├─ Base\
 │   └─ WindowsBase.vhdx       ← 読み取り専用
 └─ Diff\
     ├─ Lab01.vhdx
     ├─ Lab02.vhdx
     └─ Lab03.vhdx
```

親VHDXは完成後に読み取り専用にし、実験はすべて差分VHDX側で行うのが安全です。

[1]: https://learn.microsoft.com/ja-jp/powershell/module/hyper-v/new-vhd?view=windowsserver2025-ps "New-VHD (Hyper-V)"
[2]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/?view=windowsserver2025-ps "Hyper-V Module"
[3]: https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg252579%28v%3Dws.11%29 "Create vdisk"
[4]: https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/attach-vdisk "attach vdisk"
[5]: https://learn.microsoft.com/ja-jp/powershell/module/hyper-v/set-vhd?view=windowsserver2025-ps "Set-VHD (Hyper-V)"
[6]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/merge-vhd?view=windowsserver2025-ps "Merge-VHD (Hyper-V)"
