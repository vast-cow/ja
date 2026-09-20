---
pubDatetime: 2026-06-15T12:38:48+09:00
title: "PDFの表示内容と印刷内容をそろえるためのOCG設定ツール"
description: "PDFを画面で見たときと、印刷したときで内容が異なってしまうことがあります。 この原因の一つが、PDFに含まれる OCG、つまり表示・非表示を切り替えられるレイヤー情報です。 このツールは、OCGの影響によってPDFの表示時と印刷時の内容がずれてしまう場合に、印刷時の内容を画面表示と同じ状態に近づけ…"
---

PDFを画面で見たときと、印刷したときで内容が異なってしまうことがあります。
この原因の一つが、PDFに含まれる **OCG**、つまり表示・非表示を切り替えられるレイヤー情報です。

このツールは、OCGの影響によってPDFの表示時と印刷時の内容がずれてしまう場合に、**印刷時の内容を画面表示と同じ状態に近づける**ためのものです。

## 目的

このツールの目的は、PDF内の各レイヤーについて、表示・印刷・書き出し時の状態をそろえることです。

通常、PDFでは画面表示では見えているレイヤーが、印刷時には非表示になることがあります。
その結果、画面では問題なく見えている内容が、印刷すると消えてしまったり、逆に画面では見えない内容が印刷されてしまったりする場合があります。

このツールを使うことで、PDFの現在のレイヤー状態をもとに、印刷時や書き出し時にも同じ状態が使われるように設定できます。

## 主な用途

このツールは、次のような場合に役立ちます。

* PDFを画面で確認した内容どおりに印刷したい場合
* 印刷すると一部の文字、図、注記、背景などが消えてしまう場合
* PDF内のレイヤー設定が原因で、表示結果と印刷結果が一致しない場合
* OCGを含むPDFを、より安定して出力できる状態にしたい場合

## 使い方

基本的な使い方は、入力PDFと出力PDFを指定して実行します。

```bash
python sync_pdf_ocg_print_state.py input.pdf output.pdf
```

このコマンドを実行すると、`input.pdf` のOCG設定を確認し、表示時の状態をもとに印刷・表示・書き出し用の状態を設定したPDFを `output.pdf` として保存します。

## レイヤーを確認する

PDFにどのようなOCGレイヤーが含まれているかを確認したい場合は、次のように実行します。

```bash
python sync_pdf_ocg_print_state.py input.pdf --list-layers
```

このコマンドでは、PDF内のレイヤー番号とレイヤー名が表示されます。
PDFにOCGレイヤーが含まれているかを事前に確認したいときに便利です。

## 実際に変更せず確認する

PDFを書き換える前に、どのレイヤーがどの状態に設定される予定かを確認したい場合は、`--dry-run` を使います。

```bash
python sync_pdf_ocg_print_state.py input.pdf --dry-run
```

この場合、出力PDFは作成されません。
どのレイヤーがONまたはOFFとして扱われるかだけを確認できます。

## 詳細情報を表示する

処理時に、元のPDFのレイヤー状態に関する情報を表示したい場合は、`-v` または `--verbose` を追加します。

```bash
python sync_pdf_ocg_print_state.py input.pdf output.pdf --verbose
```

これにより、PDF内で基本状態として扱われているレイヤー設定の概要を確認できます。

## 注意点

このツールは、PDF内にOCGレイヤーが含まれている場合に効果があります。
OCGが含まれていないPDFでは、処理対象となるレイヤーがないため使用できません。

また、元のPDFを直接上書きするのではなく、別名の出力PDFを指定して保存する使い方が推奨されます。
処理前のPDFを残しておくことで、必要に応じて元の状態に戻すことができます。

## まとめ

このツールは、PDFの画面表示と印刷結果がOCGの影響で異なってしまう場合に、表示時の状態を印刷や書き出しにも反映させるためのシンプルな補助ツールです。

PDFを見たままの状態で印刷したい場合や、レイヤーによる出力差を減らしたい場合に利用できます。

```python
#!/usr/bin/env python3
"""sync_pdf_ocg_print_state.py

Set OCG Usage state metadata for all Optional Content Groups.

Default behavior:
- Resolve each OCG effective BaseState from OCProperties / get_layer(-1).
- Write that state to:
    /Usage << /Print << /PrintState ... >> /View << /ViewState ... >> /Export << /ExportState ... >> >>
- Applies to all OCGs by default; layer selection options are intentionally removed.
"""

from __future__ import annotations

import argparse
from pathlib import Path
from typing import Optional

try:
    import pymupdf as fitz  # type: ignore
except ImportError:  # pragma: no cover
    import fitz  # type: ignore


def _normalize_state(raw: str | None) -> str:
    if not raw:
        return "/ON"
    s = raw.strip()
    return s if s.startswith("/") else f"/{s}"


def _parse_refs(raw: str | None) -> set[int]:
    if not raw:
        return set()
    out: set[int] = set()
    parts = raw.replace("\r", " ").replace("\n", " ").split()
    for i in range(0, len(parts) - 2, 3):
        if parts[i + 1] == "0" and parts[i + 2] == "R" and parts[i].isdigit():
            out.add(int(parts[i]))
    return out


def _get_catalog_xref(doc: fitz.Document) -> Optional[int]:
    try:
        c = doc.pdf_catalog()
        return int(c) if c else None
    except Exception:
        for xref in range(1, doc.xref_length()):
            try:
                obj = doc.xref_object(xref, compressed=False)
            except Exception:
                continue
            if "/Type /Catalog" in obj or "/Type/Catalog" in obj:
                return xref
    return None


def _get_basestate_map(
    doc: fitz.Document,
    ocg_xrefs: set[int],
) -> tuple[str, set[int], set[int]]:
    """Return (base_state, on_set, off_set) for effective BaseState evaluation."""
    base_state = "/ON"
    on_set: set[int] = set()
    off_set: set[int] = set()

    if hasattr(doc, "get_layer"):
        try:
            cfg = doc.get_layer(-1)
            if isinstance(cfg, dict):
                raw_base = cfg.get("basestate") or cfg.get("base_state") or cfg.get("BaseState")
                if raw_base:
                    base_state = _normalize_state(str(raw_base).strip("'\""))
                on_raw = cfg.get("on") or cfg.get("ON") or []
                off_raw = cfg.get("off") or cfg.get("OFF") or []
                on_set = {
                    int(v) for v in on_raw if isinstance(v, int) or (isinstance(v, str) and v.isdigit())
                }
                off_set = {
                    int(v) for v in off_raw if isinstance(v, int) or (isinstance(v, str) and v.isdigit())
                }
                return base_state, on_set & ocg_xrefs, off_set & ocg_xrefs
        except Exception:
            pass

    catalog = _get_catalog_xref(doc)
    if not catalog:
        return base_state, on_set, off_set

    try:
        t_base, v_base = doc.xref_get_key(catalog, "OCProperties/D/BaseState")
        if t_base != "null" and v_base:
            base_state = _normalize_state(v_base)
        t_on, v_on = doc.xref_get_key(catalog, "OCProperties/D/ON")
        t_off, v_off = doc.xref_get_key(catalog, "OCProperties/D/OFF")
        if t_on != "null":
            on_set = _parse_refs(v_on)
        if t_off != "null":
            off_set = _parse_refs(v_off)
    except Exception:
        pass

    return base_state, (on_set & ocg_xrefs), (off_set & ocg_xrefs)


def _is_ocg_on_by_basestate(
    xref: int,
    base_state: str,
    on_set: set[int],
    off_set: set[int],
) -> bool:
    if base_state == "/OFF":
        return xref in on_set
    return xref not in off_set


def _collect_ocgs(doc: fitz.Document) -> dict[int, str]:
    ocgs: dict[int, str] = {}

    if hasattr(doc, "get_ocgs"):
        try:
            raw = doc.get_ocgs()
            for xref, info in raw.items():
                if isinstance(info, dict):
                    ocgs[int(xref)] = str(info.get("name", f"(xref {int(xref)})"))
        except Exception:
            ocgs = {}

    if ocgs:
        return ocgs

    for xref in range(1, doc.xref_length()):
        try:
            obj = doc.xref_object(xref, compressed=False)
        except Exception:
            continue
        if "/Type /OCG" not in obj and "/Type/OCG" not in obj:
            continue
        marker = "/Name"
        pos = obj.find(marker)
        if pos < 0:
            ocgs[xref] = f"(xref {xref})"
            continue
        start = obj.find("(", pos)
        if start < 0:
            ocgs[xref] = f"(xref {xref})"
            continue
        end = obj.find(")", start + 1)
        ocgs[xref] = obj[start + 1 : end] if end > start else f"(xref {xref})"
    return ocgs


def _ensure_usage_dict(doc: fitz.Document, ocg_xref: int) -> int:
    usage_typ, usage_val = doc.xref_get_key(ocg_xref, "Usage")

    if usage_typ == "xref" and usage_val:
        return int(usage_val.split(" ")[0])

    if usage_typ in {"null", "none"}:
        doc.xref_set_key(ocg_xref, "Usage", "<<>>")
        return ocg_xref

    if usage_typ:
        return ocg_xref

    doc.xref_set_key(ocg_xref, "Usage", "<<>>")
    return ocg_xref


def _set_usage_states(
    doc: fitz.Document,
    ocg_xref: int,
    *,
    state: str,
) -> None:
    target = _ensure_usage_dict(doc, ocg_xref)
    doc.xref_set_key(target, "Print/PrintState", state)
    doc.xref_set_key(target, "View/ViewState", state)
    doc.xref_set_key(target, "Export/ExportState", state)


def _list_layers(ocgs: dict[int, str]) -> None:
    if not ocgs:
        print("No OCG layers found.")
        return
    print("xref\tname")
    for xref in sorted(ocgs):
        print(f"{xref}\t{ocgs[xref]}")


def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(
        description=(
            "Apply OCG Usage state metadata to all layers. "
            "Default mode derives state from BaseState and applies Print/View/Export states."
        )
    )
    p.add_argument("input", help="Input PDF")
    p.add_argument("output", nargs="?", help="Output PDF")
    p.add_argument(
        "--list-layers",
        action="store_true",
        help="List OCG xref and name, then exit",
    )
    p.add_argument(
        "--dry-run",
        action="store_true",
        help="Do not write output; show planned state for each OCG",
    )
    p.add_argument(
        "-v",
        "--verbose",
        action="store_true",
        help="Show effective BaseState summary",
    )
    return p


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)

    if not args.output and not (args.list_layers or args.dry_run):
        raise SystemExit("output is required unless --list-layers or --dry-run is used")

    doc = fitz.open(Path(args.input))
    try:
        ocgs = _collect_ocgs(doc)

        if args.list_layers:
            _list_layers(ocgs)
            return 0

        if not ocgs:
            raise SystemExit("No OCG layers found.")

        base_state, base_on, base_off = _get_basestate_map(doc, set(ocgs.keys()))
        if args.verbose:
            print(f"BaseState: {base_state}, ON refs: {sorted(base_on)}, OFF refs: {sorted(base_off)}")

        if args.dry_run:
            for xref in sorted(ocgs.keys()):
                state = "/ON" if _is_ocg_on_by_basestate(xref, base_state, base_on, base_off) else "/OFF"
                print(f"Would update xref {xref} ({ocgs[xref]}) state = {state} for Print/View/Export")
            return 0

        for xref in sorted(ocgs.keys()):
            state = "/ON" if _is_ocg_on_by_basestate(xref, base_state, base_on, base_off) else "/OFF"
            _set_usage_states(doc, xref, state=state)

        doc.save(Path(args.output), garbage=4, deflate=True, clean=True, incremental=False)
        print(f"Wrote: {Path(args.output)}")
        return 0
    finally:
        doc.close()


if __name__ == "__main__":
    raise SystemExit(main())
```
