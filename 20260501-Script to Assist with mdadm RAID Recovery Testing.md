---
pubDatetime: 2026-05-01T13:24:23+09:00
title: "mdadm RAID復旧テスト支援スクリプト"
description: "目的、主な機能、使い方の概要を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

このスクリプトは、Linux の `mdadm` を使って RAID アレイの復旧候補を安全に試すためのツールです。主な目的は、元のディスクを直接書き換えずに、複数の RAID 構成パターンを試し、読み取り専用でマウントできる組み合わせを見つけることです。

## 目的

RAID アレイが壊れた、または構成情報が失われた場合、ディスクの順序、RAID レベル、チャンクサイズ、メタデータ形式、レイアウトなどを正しく推定する必要があります。

しかし、誤った `mdadm --create` を元ディスクに対して実行すると、データを破損する危険があります。

このスクリプトはそのリスクを下げるために、各ディスクの上に `dm-snapshot` による一時的なオーバーレイを作成し、そのオーバーレイデバイスに対して `mdadm --create` を試します。元のディスクは直接変更されません。

## 主な機能

このスクリプトは、指定された候補を組み合わせながら RAID アレイを作成し、ファイルシステムとしてマウントできるかを確認します。

主な機能は次の通りです。

* 元ディスクを読み取り専用に設定
* 各試行ごとに `dm-snapshot` オーバーレイを作成
* ディスク順序の組み合わせを自動生成
* 特定のディスクを特定スロットに固定
* 欠損ディスクを `missing` として試行
* `metadata`、`chunk`、`layout`、`data-offset` の候補を試行
* 読み取り専用マウントの成否を確認
* 成功した組み合わせを `successes.csv` に保存
* すべての試行結果を `results.csv` に保存
* 中断後の再開に対応

## 使い方の概要

基本的には、復旧対象のディスク、RAID レベル、RAID デバイス数、試したいパラメータを指定して実行します。

例:

```bash
sudo ./mdadm_try_mount.py \
  --origins /dev/sdb /dev/sdc /dev/sdd /dev/sde \
  --level 5 \
  --raid-devices 4 \
  --metadata 1.2 1.0 \
  --chunks 64K 128K 256K 512K \
  --layouts left-symmetric right-symmetric \
  --workdir /root/mdadm-try
```

この例では、4 台構成の RAID5 として、複数のメタデータ形式、チャンクサイズ、レイアウトを試します。

## 固定スロットの指定

一部のディスクの位置が分かっている場合は、`--fixed-slot` を使ってスロットを固定できます。

```bash
sudo ./mdadm_try_mount.py \
  --origins /dev/sdb /dev/sdc /dev/sdd /dev/sde \
  --fixed-slot 0=/dev/sdb \
  --level 5 \
  --raid-devices 4
```

この場合、`/dev/sdb` は RAID のスロット 0 として固定され、残りのディスクだけが並べ替えの対象になります。

## 欠損ディスクを含めて試す

RAID5 や RAID6 では、1 台または複数台のディスクが欠けた状態でも復旧できる場合があります。

そのような場合は、`--include-missing` を指定します。

```bash
sudo ./mdadm_try_mount.py \
  --origins /dev/sdb /dev/sdc /dev/sdd \
  --level 5 \
  --raid-devices 4 \
  --include-missing \
  --max-missing 1
```

この例では、4 台構成だった RAID5 のうち、手元に 3 台しかないケースを想定しています。

## 結果の確認

実行後、作業ディレクトリに主に次のファイルが作成されます。

```text
results.csv
successes.csv
logs/
samples/
```

`results.csv` にはすべての試行結果が記録されます。

`successes.csv` には、読み取り専用マウントに成功した組み合わせだけが記録されます。

成功した場合は、`samples/` 以下にマウント先で見つかったファイル一覧も保存されます。これにより、正しい構成かどうかを確認しやすくなります。

## 再開機能

大量の組み合わせを試す場合、処理に時間がかかることがあります。中断後に再開したい場合は、次のように指定できます。

```bash
sudo ./mdadm_try_mount.py \
  --resume-from-results \
  ...
```

既存の `results.csv` を読み取り、最後に記録された試行番号の次から再開します。

## 安全に使うための注意点

このスクリプトは安全性を重視して設計されていますが、RAID 復旧作業自体は慎重に行う必要があります。

特に重要なのは、元ディスクに対して直接 `mdadm --create` を実行しないことです。このスクリプトはオーバーレイデバイスだけを対象にする設計ですが、指定するデバイス名や作業ディレクトリは必ず確認してください。

また、作業ディレクトリには COW ファイルが作られるため、十分な空き容量がある通常のディスク上を指定する必要があります。`tmpfs` や `ramfs` は拒否されるようになっています。

## まとめ

このスクリプトは、RAID 構成が不明な状況で、候補を機械的かつ安全に試すための補助ツールです。

特に、次のような場面で役立ちます。

* RAID ディスクの順序が分からない
* チャンクサイズやレイアウトが不明
* 一部のディスクが欠損している
* 読み取り専用で復旧可能な構成を探したい
* 試行結果を CSV とログで管理したい

RAID 復旧の最初の調査段階で、危険な直接操作を避けながら、正しい構成候補を絞り込むために使うツールです。

```python
#!/usr/bin/env python3
"""
mdadm_try_mount.py

Safely test mdadm --create parameter combinations on dm-snapshot overlays,
then collect combinations that successfully mount read-only.

Features:
  - dm-snapshot overlay per trial
  - --fixed-slot SLOT=DEVICE
  - tqdm progress bar with estimated total trials
  - read-only mount checks
  - success extraction to successes.csv

IMPORTANT:
  Do NOT run mdadm --create directly on original disks.
  This script passes only /dev/mapper/mdtry_* overlay devices to mdadm --create.
"""

import argparse
import csv
import itertools
import math
import os
import re
import shlex
import shutil
import subprocess
import sys
import time
from pathlib import Path
from typing import Dict, Iterable, Iterator, List, Optional, Sequence, Tuple

from tqdm import tqdm


# -----------------------------
# Basic command helpers
# -----------------------------

def run(
    cmd: Sequence[str],
    check: bool = False,
    capture: bool = True,
    timeout: Optional[int] = None,
    quiet: bool = False,
) -> subprocess.CompletedProcess:
    if not quiet:
        tqdm.write("+ " + " ".join(shlex.quote(str(x)) for x in cmd))

    return subprocess.run(
        list(map(str, cmd)),
        check=check,
        text=True,
        stdout=subprocess.PIPE if capture else None,
        stderr=subprocess.PIPE if capture else None,
        timeout=timeout,
    )


def require_root() -> None:
    if os.geteuid() != 0:
        raise SystemExit("ERROR: please run as root.")


def require_tools(tools: Sequence[str]) -> None:
    missing = [tool for tool in tools if shutil.which(tool) is None]
    if missing:
        raise SystemExit(f"ERROR: required command(s) not found: {', '.join(missing)}")


def write_text(path: Path, text: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(text or "", encoding="utf-8", errors="replace")


def safe_name(dev: str) -> str:
    return re.sub(r"[^A-Za-z0-9_.-]+", "_", dev.strip("/"))


def device_exists(dev: str) -> bool:
    return Path(dev).exists()


# -----------------------------
# Block / filesystem helpers
# -----------------------------

def filesystem_type_for_path(path: Path) -> str:
    path.mkdir(parents=True, exist_ok=True)

    cp = run(
        ["findmnt", "-n", "-o", "FSTYPE", "-T", str(path)],
        check=True,
        quiet=True,
    )
    return cp.stdout.strip()


def ensure_workdir_not_tmpfs(workdir: Path) -> None:
    fstype = filesystem_type_for_path(workdir)

    if fstype in {"tmpfs", "ramfs"}:
        raise SystemExit(
            f"ERROR: workdir={workdir} is {fstype}.\n"
            "This is unsafe because COW files will consume RAM.\n"
            "Specify a --workdir path on a normal physical disk."
        )

    tqdm.write(f"workdir filesystem: {fstype}")


def get_sectors(dev: str) -> int:
    cp = run(["blockdev", "--getsz", dev], check=True, quiet=True)
    return int(cp.stdout.strip())


def set_readonly(dev: str) -> None:
    run(["blockdev", "--setro", dev], check=False, quiet=True)

    cp = run(["blockdev", "--getro", dev], check=True, quiet=True)
    if cp.stdout.strip() != "1":
        raise RuntimeError(f"{dev} could not be set to read-only")

    tqdm.write(f"{dev}: read-only")


# -----------------------------
# mdadm / dmsetup / loop helpers
# -----------------------------

def stop_md(mddev: str, quiet: bool = True) -> None:
    run(["mdadm", "--stop", mddev], check=False, quiet=quiet)


def make_cow_file(path: Path, size: str, quiet: bool = True) -> str:
    path.parent.mkdir(parents=True, exist_ok=True)

    run(["truncate", "-s", size, str(path)], check=True, quiet=quiet)

    cp = run(["losetup", "-f", "--show", str(path)], check=True, quiet=quiet)
    loopdev = cp.stdout.strip()

    if not loopdev:
        raise RuntimeError(f"losetup failed for {path}")

    return loopdev


def detach_loop(loopdev: str, quiet: bool = True) -> None:
    if loopdev:
        run(["losetup", "-d", loopdev], check=False, quiet=quiet)


def create_snapshot_overlay(
    origin: str,
    cow_loop: str,
    mapper_name: str,
    snapshot_chunk_sectors: int,
    quiet: bool = True,
) -> str:
    sectors = get_sectors(origin)
    table = f"0 {sectors} snapshot {origin} {cow_loop} N {snapshot_chunk_sectors}"

    run(["dmsetup", "create", mapper_name, "--table", table], check=True, quiet=quiet)

    return f"/dev/mapper/{mapper_name}"


def remove_mapper(name: str, quiet: bool = True) -> None:
    if name:
        run(["dmsetup", "remove", name], check=False, quiet=quiet)


def dm_status(names: Sequence[str]) -> str:
    out: List[str] = []

    for name in names:
        cp = run(["dmsetup", "status", name], check=False, quiet=True)
        text = ((cp.stdout or "") + (cp.stderr or "")).strip()
        out.append(f"{name}: {text}")

    return "\n".join(out)


# -----------------------------
# Probe / mount helpers
# -----------------------------

def blkid_probe(dev: str) -> str:
    cp = run(["blkid", "-p", dev], check=False, quiet=True)
    return ((cp.stdout or "") + (cp.stderr or "")).strip()


def file_probe(dev: str) -> str:
    cp = run(["file", "-s", dev], check=False, quiet=True)
    return ((cp.stdout or "") + (cp.stderr or "")).strip()


def detect_fs(blkid_text: str, file_text: str) -> Optional[str]:
    combined = f"{blkid_text}\n{file_text}".lower()

    m = re.search(r'type="([^"]+)"', blkid_text, re.IGNORECASE)
    if m:
        return m.group(1).lower()

    for fs in [
        "ext4",
        "ext3",
        "ext2",
        "xfs",
        "btrfs",
        "vfat",
        "ntfs",
        "exfat",
    ]:
        if fs in combined:
            return fs

    if "crypto_luks" in combined or "luks" in combined:
        return "luks"

    if "lvm2_member" in combined or "lvm" in combined:
        return "lvm"

    return None


def mount_options_for_fs(fs_type: str) -> Tuple[List[str], str]:
    fs_type = fs_type.lower()

    if fs_type in {"ext4", "ext3", "ext2"}:
        return ["-o", "ro,noload"], "ro,noload"

    if fs_type == "xfs":
        return ["-o", "ro,norecovery"], "ro,norecovery"

    if fs_type in {"btrfs", "vfat", "ntfs", "exfat"}:
        return ["-o", "ro"], "ro"

    return ["-o", "ro"], "ro"


def try_mount(mddev: str, mnt: Path, fs_type: Optional[str]) -> Tuple[bool, str, str]:
    mnt.mkdir(parents=True, exist_ok=True)

    if not fs_type:
        return False, "", "unknown filesystem"

    if fs_type in {"luks", "lvm"}:
        return False, "", f"{fs_type}: is not a direct mount target"

    opts, opts_text = mount_options_for_fs(fs_type)

    cp = run(
        ["mount", "-t", fs_type] + opts + [mddev, str(mnt)],
        check=False,
        timeout=30,
        quiet=True,
    )
    out = ((cp.stdout or "") + (cp.stderr or "")).strip()

    if cp.returncode == 0:
        return True, opts_text, out

    cp2 = run(
        ["mount", "-o", "ro", mddev, str(mnt)],
        check=False,
        timeout=30,
        quiet=True,
    )
    out2 = ((cp2.stdout or "") + (cp2.stderr or "")).strip()

    if cp2.returncode == 0:
        return True, "ro(auto)", out2

    return False, opts_text, f"{out}\n{out2}".strip()


def unmount(mnt: Path, quiet: bool = True) -> None:
    run(["umount", str(mnt)], check=False, quiet=quiet)


def list_sample_files(mnt: Path, limit: int = 100) -> List[str]:
    samples: List[str] = []

    try:
        for root, dirs, filenames in os.walk(mnt):
            dirs.sort()
            filenames.sort()

            for filename in filenames:
                p = Path(root) / filename
                rel = p.relative_to(mnt)
                samples.append(str(rel))

                if len(samples) >= limit:
                    return samples

    except Exception as exc:
        samples.append(f"ERROR listing files: {exc}")

    return samples


# -----------------------------
# fixed-slot handling
# -----------------------------

def parse_fixed_slots(
    fixed_slot_args: Sequence[str],
    origins: Sequence[str],
    raid_devices: int,
) -> Dict[int, str]:
    """
    Parse:
      --fixed-slot 2=/dev/sdk
      --fixed-slot 3:/dev/sdm

    Returns:
      {2: "/dev/sdk", 3: "/dev/sdm"}
    """
    fixed: Dict[int, str] = {}
    origin_set = set(origins)

    for item in fixed_slot_args:
        if "=" in item:
            left, right = item.split("=", 1)
        elif ":" in item:
            left, right = item.split(":", 1)
        else:
            raise SystemExit(
                f"ERROR: invalid --fixed-slot format: {item}\n"
                "Use SLOT=DEVICE, e.g. --fixed-slot 3=/dev/sdm"
            )

        left = left.strip()
        right = str(Path(right.strip()))

        if not left.isdigit():
            raise SystemExit(f"ERROR: fixed slot is not numeric: {item}")

        slot = int(left)

        if slot < 0 or slot >= raid_devices:
            raise SystemExit(
                f"ERROR: fixed slot out of range: slot={slot}, raid_devices={raid_devices}"
            )

        if not device_exists(right):
            raise SystemExit(f"ERROR: fixed-slot device not found: {right}")

        if right not in origin_set:
            raise SystemExit(
                f"ERROR: fixed-slot device {right} is not included in --origins.\n"
                "Add it to --origins as well."
            )

        if slot in fixed and fixed[slot] != right:
            raise SystemExit(
                f"ERROR: slot {slot} is assigned multiple devices: {fixed[slot]} and {right}"
            )

        if right in fixed.values():
            prev = [s for s, d in fixed.items() if d == right][0]
            raise SystemExit(
                f"ERROR: device {right} is assigned to multiple slots: {prev} and {slot}"
            )

        fixed[slot] = right

    return fixed


def fixed_slots_to_labels(
    fixed_slots: Dict[int, str],
    dev_to_label: Dict[str, str],
) -> Dict[int, str]:
    return {slot: dev_to_label[dev] for slot, dev in fixed_slots.items()}


# -----------------------------
# Combination generators
# -----------------------------

def iter_orders(
    dev_labels: Sequence[str],
    raid_devices: int,
    fixed_slots: Dict[int, str],
    include_missing: bool,
    max_missing: int,
    deduplicate: bool = True,
) -> Iterator[Tuple[str, ...]]:
    """
    Generate slot arrays of length raid_devices.

    fixed_slots:
      {slot_index: label}

    If fixed_slots exist, only unknown slots are permuted.

    If include_missing:
      unknown slots may also be filled with "missing", up to max_missing total.
      Fixed slots are never replaced with missing.
    """
    if raid_devices <= 0:
        raise SystemExit("ERROR: raid_devices must be > 0")

    all_slots = list(range(raid_devices))
    fixed_slot_set = set(fixed_slots.keys())
    unknown_slots = [s for s in all_slots if s not in fixed_slot_set]

    fixed_labels = set(fixed_slots.values())
    remaining_labels = [x for x in dev_labels if x not in fixed_labels]

    if len(fixed_labels) != len(fixed_slots):
        raise SystemExit("ERROR: duplicate fixed slot labels detected")

    if len(remaining_labels) > len(unknown_slots):
        raise SystemExit(
            "ERROR: unfixed origin devices are more than available unknown slots.\n"
            f"remaining_labels={len(remaining_labels)}, unknown_slots={len(unknown_slots)}"
        )

    seen = set()

    def emit(order: Tuple[str, ...]) -> Optional[Tuple[str, ...]]:
        if not deduplicate:
            return order

        if order in seen:
            return None

        seen.add(order)
        return order

    def build_order_for_unknown(values: Sequence[str]) -> Tuple[str, ...]:
        arr: List[Optional[str]] = [None] * raid_devices

        for slot, label in fixed_slots.items():
            arr[slot] = label

        for slot, value in zip(unknown_slots, values):
            arr[slot] = value

        unresolved = [i for i, v in enumerate(arr) if v is None]
        if unresolved:
            raise RuntimeError(f"internal error: unresolved slots: {unresolved}")

        return tuple(str(x) for x in arr)

    # No-missing case.
    if len(remaining_labels) == len(unknown_slots):
        for perm in itertools.permutations(remaining_labels):
            order = build_order_for_unknown(perm)
            out = emit(order)
            if out is not None:
                yield out

    # Missing cases.
    if include_missing:
        if max_missing < 0:
            raise SystemExit("ERROR: max_missing must be >= 0")

        max_missing_effective = min(max_missing, len(unknown_slots))

        for missing_count in range(1, max_missing_effective + 1):
            value_count = len(unknown_slots) - missing_count

            if value_count > len(remaining_labels):
                continue

            for missing_slots_local in itertools.combinations(range(len(unknown_slots)), missing_count):
                missing_slots_local_set = set(missing_slots_local)

                for chosen_labels in itertools.permutations(remaining_labels, value_count):
                    values: List[str] = []
                    it = iter(chosen_labels)

                    for local_idx in range(len(unknown_slots)):
                        if local_idx in missing_slots_local_set:
                            values.append("missing")
                        else:
                            values.append(next(it))

                    order = build_order_for_unknown(values)
                    out = emit(order)
                    if out is not None:
                        yield out


def iter_combos(
    orders: Iterable[Tuple[str, ...]],
    metadata: Sequence[str],
    chunks: Sequence[str],
    layouts: Sequence[str],
    data_offsets: Sequence[str],
) -> Iterator[Tuple[Tuple[str, ...], str, str, str, Optional[str]]]:
    offsets: Sequence[Optional[str]]

    if data_offsets:
        offsets = list(data_offsets)
    else:
        offsets = [None]

    for order in orders:
        for meta in metadata:
            for chunk in chunks:
                for layout in layouts:
                    for data_offset in offsets:
                        yield order, meta, chunk, layout, data_offset


# -----------------------------
# Count estimation for tqdm
# -----------------------------

def nperm(n: int, r: int) -> int:
    if r < 0 or r > n:
        return 0
    return math.factorial(n) // math.factorial(n - r)


def estimate_order_count(
    dev_labels: Sequence[str],
    raid_devices: int,
    fixed_slots: Dict[int, str],
    include_missing: bool,
    max_missing: int,
) -> int:
    unknown_slots_count = raid_devices - len(fixed_slots)
    fixed_labels = set(fixed_slots.values())
    remaining_labels_count = len([x for x in dev_labels if x not in fixed_labels])

    if unknown_slots_count < 0:
        return 0

    total = 0

    # No-missing case.
    if remaining_labels_count == unknown_slots_count:
        total += math.factorial(remaining_labels_count)

    # Missing cases.
    if include_missing:
        max_missing_effective = min(max_missing, unknown_slots_count)

        for missing_count in range(1, max_missing_effective + 1):
            value_count = unknown_slots_count - missing_count

            if value_count > remaining_labels_count:
                continue

            missing_slot_choices = math.comb(unknown_slots_count, missing_count)
            device_orders = nperm(remaining_labels_count, value_count)

            total += missing_slot_choices * device_orders

    return total


def estimate_combo_count(
    dev_labels: Sequence[str],
    raid_devices: int,
    fixed_slots: Dict[int, str],
    include_missing: bool,
    max_missing: int,
    metadata: Sequence[str],
    chunks: Sequence[str],
    layouts: Sequence[str],
    data_offsets: Sequence[str],
    limit: int,
) -> int:
    order_count = estimate_order_count(
        dev_labels=dev_labels,
        raid_devices=raid_devices,
        fixed_slots=fixed_slots,
        include_missing=include_missing,
        max_missing=max_missing,
    )

    offset_count = len(data_offsets) if data_offsets else 1

    total = (
        order_count
        * len(metadata)
        * len(chunks)
        * len(layouts)
        * offset_count
    )

    if limit and limit > 0:
        total = min(total, limit)

    return total


# -----------------------------
# CSV helpers
# -----------------------------

RESULT_FIELDS = [
    "try_id",
    "mount_success",
    "fs_type",
    "mount_options",
    "level",
    "raid_devices",
    "metadata",
    "chunk",
    "layout",
    "data_offset",
    "order",
    "fixed_slots",
    "blkid",
    "file",
    "mount_message",
    "sample_file",
]


def sanitize_csv_text(text: str) -> str:
    return (text or "").replace("\n", " ").replace("\r", " ").strip()


def format_fixed_slots(fixed_slots: Dict[int, str]) -> str:
    return " ".join(f"{slot}={label}" for slot, label in sorted(fixed_slots.items()))


def parse_try_id(value: str) -> int:
    """Parse try_id values such as 123, 000123, or 000123.

    The try_id in the first column of results.csv is a zero-padded string such as 000001, but
    the CLI also accepts plain numbers such as 123.
    """
    text = str(value).strip()

    if not text.isdigit():
        raise argparse.ArgumentTypeError(
            f"try_id must be numeric, got: {value!r}"
        )

    n = int(text)

    if n < 0:
        raise argparse.ArgumentTypeError("try_id must be >= 0")

    return n


def latest_try_id_from_results_csv(path: Path) -> int:
    """Return the largest numeric try_id in results.csv, or 0 if unavailable."""
    if not path.exists():
        return 0

    latest = 0

    with path.open("r", newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)

        if not reader.fieldnames or "try_id" not in reader.fieldnames:
            return 0

        for row in reader:
            raw = (row.get("try_id") or "").strip()
            if raw.isdigit():
                latest = max(latest, int(raw))

    return latest


# -----------------------------
# Args
# -----------------------------

def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description=(
            "Safely test mdadm --create combinations on dm-snapshot overlays "
            "and collect read-only mount successes."
        )
    )

    parser.add_argument(
        "--origins",
        nargs="+",
        required=True,
        help="Origin devices. Example: /dev/sdb /dev/sdc /dev/sdd",
    )

    parser.add_argument(
        "--fixed-slot",
        action="append",
        default=[],
        help=(
            "Fix known slots. Can be specified multiple times."
            "Example: --fixed-slot 2=/dev/sdk --fixed-slot 3=/dev/sdm"
        ),
    )

    parser.add_argument(
        "--level",
        required=True,
        help="RAID level. Examples: 1, 5, 6, 10, raid5, raid6",
    )

    parser.add_argument(
        "--raid-devices",
        type=int,
        required=True,
        help="Total number of devices in the original RAID array",
    )

    parser.add_argument(
        "--metadata",
        nargs="+",
        default=["1.2"],
        help="Metadata versions to try. Example: 1.2 1.0 0.90",
    )

    parser.add_argument(
        "--chunks",
        nargs="+",
        default=["512K"],
        help="Chunk sizes to try. Example: 64K 128K 256K 512K 1024K",
    )

    parser.add_argument(
        "--layouts",
        nargs="+",
        default=["left-symmetric"],
        help=(
            "Layouts to try. RAID5/6 examples: "
            "left-symmetric left-asymmetric right-symmetric right-asymmetric。"
            "Specify none if unnecessary, such as for RAID1."
        ),
    )

    parser.add_argument(
        "--data-offsets",
        nargs="*",
        default=[],
        help="Data offsets to try. Example: 264192s 2048K. If empty, no data offset is specified.",
    )

    parser.add_argument(
        "--mddev",
        default="/dev/md127",
        help="md device used for trials. default: /dev/md127",
    )

    parser.add_argument(
        "--workdir",
        default="/root/mdadm-try",
        help="Working directory. tmpfs/ramfs is rejected.",
    )

    parser.add_argument(
        "--cow-size",
        default="4G",
        help="COW size for each overlay. default: 4G",
    )

    parser.add_argument(
        "--snapshot-chunk-sectors",
        type=int,
        default=1024,
        help="dm-snapshot chunk size in sectors. default: 1024 = 512KiB",
    )

    parser.add_argument(
        "--include-missing",
        action="store_true",
        help="Also try combinations that include missing slots",
    )

    parser.add_argument(
        "--max-missing",
        type=int,
        default=1,
        help="Maximum number of missing slots. Use 1 for RAID5, 2 for RAID6, etc.",
    )

    parser.add_argument(
        "--limit",
        type=int,
        default=0,
        help="Maximum number of trials. 0 means unlimited.",
    )

    parser.add_argument(
        "--resume-from",
        type=parse_try_id,
        default=0,
        metavar="TRY_ID",
        help=(
            "Resume from the try_id in the first column of results.csv."
            "Trials up to the specified try_id are skipped, and appending resumes from the next try_id."
            "Example: --resume-from 123 or --resume-from 000123"
        ),
    )

    parser.add_argument(
        "--resume-from-results",
        action="store_true",
        help=(
            "Read the maximum try_id from the first column of the existing results.csv,"
            "then automatically resume from the next try_id."
        ),
    )

    parser.add_argument(
        "--no-setro",
        action="store_true",
        help="Do not run blockdev --setro on origin devices",
    )

    parser.add_argument(
        "--no-deduplicate-orders",
        action="store_true",
        help="Disable order deduplication. This saves memory but may increase duplicate trials.",
    )

    parser.add_argument(
        "--sample-limit",
        type=int,
        default=100,
        help="Number of sample file paths to save on successful mount. default: 100",
    )

    parser.add_argument(
        "--keep-cow-on-success",
        action="store_true",
        help=(
            "Keep COW/mapper/loop devices on successful mount."
            "For further investigation. Be careful: leaving them in place consumes disk space and memory."
        ),
    )

    parser.add_argument(
        "--no-progress",
        action="store_true",
        help="Disable tqdm progress display",
    )

    parser.add_argument(
        "--verbose-commands",
        action="store_true",
        help="Show each command verbosely. This may disrupt tqdm display.",
    )

    return parser.parse_args()


# -----------------------------
# Main
# -----------------------------

def main() -> int:
    args = parse_args()

    require_root()

    require_tools([
        "mdadm",
        "dmsetup",
        "losetup",
        "truncate",
        "blockdev",
        "blkid",
        "file",
        "mount",
        "umount",
        "findmnt",
    ])

    origins = [str(Path(x)) for x in args.origins]

    for dev in origins:
        if not device_exists(dev):
            raise SystemExit(f"ERROR: device not found: {dev}")

    if args.max_missing < 0:
        raise SystemExit("ERROR: --max-missing must be >= 0")

    if args.raid_devices <= 0:
        raise SystemExit("ERROR: --raid-devices must be > 0")

    if len(origins) > args.raid_devices:
        raise SystemExit(
            f"ERROR: origins count exceeds raid_devices: origins={len(origins)}, "
            f"raid_devices={args.raid_devices}"
        )

    workdir = Path(args.workdir)
    ensure_workdir_not_tmpfs(workdir)

    logs_dir = workdir / "logs"
    cow_dir = workdir / "cow"
    mnt_base = workdir / "mnt"
    sample_dir = workdir / "samples"

    results_csv = workdir / "results.csv"
    successes_csv = workdir / "successes.csv"

    resume_from = args.resume_from
    if args.resume_from_results:
        detected_resume_from = latest_try_id_from_results_csv(results_csv)
        resume_from = max(resume_from, detected_resume_from)

    if resume_from < 0:
        raise SystemExit("ERROR: --resume-from must be >= 0")

    for directory in [logs_dir, cow_dir, mnt_base, sample_dir]:
        directory.mkdir(parents=True, exist_ok=True)

    labels = [safe_name(dev) for dev in origins]
    dev_to_label: Dict[str, str] = dict(zip(origins, labels))
    label_to_origin: Dict[str, str] = dict(zip(labels, origins))

    fixed_slots_dev = parse_fixed_slots(
        fixed_slot_args=args.fixed_slot,
        origins=origins,
        raid_devices=args.raid_devices,
    )
    fixed_slots_label = fixed_slots_to_labels(
        fixed_slots=fixed_slots_dev,
        dev_to_label=dev_to_label,
    )

    tqdm.write("Origins:")
    for label, origin in label_to_origin.items():
        tqdm.write(f"  {label}: {origin}")

    if fixed_slots_label:
        tqdm.write("\nFixed slots:")
        for slot, label in sorted(fixed_slots_label.items()):
            tqdm.write(f"  slot {slot}: {label_to_origin[label]} ({label})")
    else:
        tqdm.write("\nFixed slots: none")

    if not args.no_setro:
        tqdm.write("\nSetting origin devices to read-only.")
        for dev in origins:
            set_readonly(dev)
    else:
        tqdm.write("\nWARNING: --no-setro was specified. Origin devices were not set to read-only.")

    deduplicate_orders = not args.no_deduplicate_orders

    order_iter = iter_orders(
        dev_labels=labels,
        raid_devices=args.raid_devices,
        fixed_slots=fixed_slots_label,
        include_missing=args.include_missing,
        max_missing=args.max_missing,
        deduplicate=deduplicate_orders,
    )

    combo_iter = iter_combos(
        orders=order_iter,
        metadata=args.metadata,
        chunks=args.chunks,
        layouts=args.layouts,
        data_offsets=args.data_offsets,
    )

    fixed_slots_text = format_fixed_slots(fixed_slots_label)

    total_combos = estimate_combo_count(
        dev_labels=labels,
        raid_devices=args.raid_devices,
        fixed_slots=fixed_slots_label,
        include_missing=args.include_missing,
        max_missing=args.max_missing,
        metadata=args.metadata,
        chunks=args.chunks,
        layouts=args.layouts,
        data_offsets=args.data_offsets,
        limit=args.limit,
    )

    tqdm.write(f"\nEstimated total tries: {total_combos}")

    if resume_from:
        tqdm.write(
            f"Resume: skipping completed try_id <= {resume_from:06d}; "
            f"next try_id is {resume_from + 1:06d}"
        )

    if total_combos == 0:
        raise SystemExit(
            "ERROR: The number of trials is 0.\n"
            "Check the combination of --origins, --fixed-slot, --raid-devices, and --include-missing."
        )

    append_results = resume_from > 0 and results_csv.exists()
    append_successes = resume_from > 0 and successes_csv.exists()

    progress_initial = min(resume_from, total_combos) if resume_from else 0

    with results_csv.open("a" if append_results else "w", newline="", encoding="utf-8") as rf, \
         successes_csv.open("a" if append_successes else "w", newline="", encoding="utf-8") as sf:

        result_writer = csv.DictWriter(rf, fieldnames=RESULT_FIELDS)
        success_writer = csv.DictWriter(sf, fieldnames=RESULT_FIELDS)

        if not append_results:
            result_writer.writeheader()
        if not append_successes:
            success_writer.writeheader()

        progress = tqdm(
            total=total_combos,
            initial=progress_initial,
            unit="try",
            dynamic_ncols=True,
            disable=args.no_progress,
            desc="mdadm trials",
        )

        try:
            for idx, (order, meta, chunk, layout, data_offset) in enumerate(combo_iter, 1):
                if args.limit and idx > args.limit:
                    tqdm.write(f"limit reached: {args.limit}")
                    break

                if resume_from and idx <= resume_from:
                    continue

                try_id = f"{idx:06d}"

                mapper_names: List[str] = []
                loopdevs: List[str] = []
                overlay_for_label: Dict[str, str] = {}

                mnt = mnt_base / f"try_{try_id}"

                blkid_text = ""
                file_text = ""
                fs_type = ""
                mount_success = False
                mount_options = ""
                mount_message = ""
                sample_file = ""

                keep_current = False
                quiet_commands = not args.verbose_commands

                try:
                    stop_md(args.mddev, quiet=quiet_commands)

                    # Create fresh overlays for this try.
                    for label, origin in label_to_origin.items():
                        cow_file = cow_dir / f"{try_id}_{label}.cow"
                        loopdev = make_cow_file(cow_file, args.cow_size, quiet=quiet_commands)
                        loopdevs.append(loopdev)

                        mapper_name = f"mdtry_{try_id}_{label}"
                        mapper_names.append(mapper_name)

                        overlay_dev = create_snapshot_overlay(
                            origin=origin,
                            cow_loop=loopdev,
                            mapper_name=mapper_name,
                            snapshot_chunk_sectors=args.snapshot_chunk_sectors,
                            quiet=quiet_commands,
                        )

                        overlay_for_label[label] = overlay_dev

                    mdadm_cmd: List[str] = [
                        "mdadm",
                        "--create",
                        args.mddev,
                        "--assume-clean",
                        "--readonly",
                        "--force",
                        f"--metadata={meta}",
                        f"--level={args.level}",
                        f"--raid-devices={args.raid_devices}",
                        f"--chunk={chunk}",
                    ]

                    if layout.lower() not in {"", "none", "null", "-"}:
                        mdadm_cmd.append(f"--layout={layout}")

                    if data_offset:
                        mdadm_cmd.append(f"--data-offset={data_offset}")

                    for slot in order:
                        if slot == "missing":
                            mdadm_cmd.append("missing")
                        else:
                            mdadm_cmd.append(overlay_for_label[slot])

                    cp_create = run(
                        mdadm_cmd,
                        check=False,
                        timeout=90,
                        quiet=quiet_commands,
                    )

                    write_text(logs_dir / f"{try_id}.mdadm.stdout.log", cp_create.stdout or "")
                    write_text(logs_dir / f"{try_id}.mdadm.stderr.log", cp_create.stderr or "")

                    if cp_create.returncode != 0:
                        mount_message = "mdadm create failed"
                    else:
                        time.sleep(2)

                        write_text(
                            logs_dir / f"{try_id}.dmstatus.after-create.log",
                            dm_status(mapper_names),
                        )

                        blkid_text = blkid_probe(args.mddev)
                        file_text = file_probe(args.mddev)

                        fs = detect_fs(blkid_text, file_text)
                        fs_type = fs or ""

                        mount_success, mount_options, mount_message = try_mount(
                            args.mddev,
                            mnt,
                            fs,
                        )

                        write_text(
                            logs_dir / f"{try_id}.dmstatus.after-mount.log",
                            dm_status(mapper_names),
                        )

                        if mount_success:
                            samples = list_sample_files(mnt, limit=args.sample_limit)
                            sample_path = sample_dir / f"{try_id}.files.txt"
                            write_text(sample_path, "\n".join(samples))
                            sample_file = str(sample_path)

                            success_log = (
                                f"ok=yes try_id={try_id} "
                                f"fs_type={fs_type or '-'} "
                                f"mount_options={mount_options or '-'} "
                                f"level={args.level} "
                                f"raid_devices={args.raid_devices} "
                                f"metadata={meta} "
                                f"chunk={chunk} "
                                f"layout={layout} "
                                f"data_offset={data_offset or '-'} "
                                f"order=\"{' '.join(order)}\" "
                                f"sample_file={sample_file}"
                            )
                            tqdm.write(success_log)

                            if args.keep_cow_on_success:
                                keep_current = True
                                tqdm.write(
                                    "WARNING: --keep-cow-on-success is set, so"
                                    "the mapper/loop/COW devices for this trial will be kept."
                                )

                except subprocess.TimeoutExpired as exc:
                    mount_message = f"timeout: {exc}"

                except Exception as exc:
                    mount_message = f"exception: {exc}"

                finally:
                    unmount(mnt, quiet=quiet_commands)

                    if not keep_current:
                        stop_md(args.mddev, quiet=quiet_commands)

                        for mapper_name in reversed(mapper_names):
                            remove_mapper(mapper_name, quiet=quiet_commands)

                        for loopdev in reversed(loopdevs):
                            detach_loop(loopdev, quiet=quiet_commands)

                        for label in labels:
                            cow_file = cow_dir / f"{try_id}_{label}.cow"
                            try:
                                cow_file.unlink(missing_ok=True)
                            except Exception:
                                pass
                    else:
                        tqdm.write(f"Kept md device: {args.mddev}")
                        tqdm.write("Kept mapper devices: " + " ".join(mapper_names))
                        tqdm.write("Kept loop devices: " + " ".join(loopdevs))

                row = {
                    "try_id": try_id,
                    "mount_success": "yes" if mount_success else "no",
                    "fs_type": fs_type,
                    "mount_options": mount_options,
                    "level": args.level,
                    "raid_devices": str(args.raid_devices),
                    "metadata": meta,
                    "chunk": chunk,
                    "layout": layout,
                    "data_offset": data_offset or "",
                    "order": " ".join(order),
                    "fixed_slots": fixed_slots_text,
                    "blkid": sanitize_csv_text(blkid_text),
                    "file": sanitize_csv_text(file_text),
                    "mount_message": sanitize_csv_text(mount_message),
                    "sample_file": sample_file,
                }

                result_writer.writerow(row)
                rf.flush()

                if mount_success:
                    success_writer.writerow(row)
                    sf.flush()

                    if args.keep_cow_on_success:
                        tqdm.write(
                            "\nStopping at the successful trial because --keep-cow-on-success is set.\n"
                            "After investigation, manually run umount / mdadm --stop / dmsetup remove / losetup -d."
                        )
                        progress.update(1)
                        break

                progress.update(1)

        finally:
            progress.close()

    tqdm.write("\nDone.")
    tqdm.write(f"All results    : {results_csv}")
    tqdm.write(f"Mount successes: {successes_csv}")

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```
