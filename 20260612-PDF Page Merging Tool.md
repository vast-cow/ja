---
pubDatetime: 2026-06-12T22:06:11+09:00
title: "PDFページ結合ツール"
description: "概要 このツールは、PDFの複数ページを各出力ページ上のグリッドレイアウトに結合します。4ページを2 × 2レイアウトに配置するなど、複数のPDFページをまとめて配置したい場合に便利です。 このツールは、ページ数の削減、配布資料の準備、概要シートの作成、PDF文書のコンパクト版の作成に使用できます。…"
---

## 概要

このツールは、PDFの複数ページを各出力ページ上のグリッドレイアウトに結合します。4ページを2 × 2レイアウトに配置するなど、複数のPDFページをまとめて配置したい場合に便利です。

このツールは、ページ数の削減、配布資料の準備、概要シートの作成、PDF文書のコンパクト版の作成に使用できます。

## 主な目的

このツールの主な目的は、**n-up PDF**を作成することです。

「N-up」とは、複数のページを1枚のシートに配置することを意味します。たとえば、2 × 2レイアウトでは、4つの元PDFページを1つの出力ページに配置します。

このツールでは、次のことができます。

* 入力PDFの全ページまたは特定のページのみを選択する
* ページを行と列に配置する
* 元ページを縮小せずに、より大きな出力ページを作成する
* 元のページサイズに収まるようにページを縮小する
* 出力ページの周囲に余白を追加する
* 配置されたページ間に間隔を追加する

## 仕組み

このツールは入力PDFを読み込み、要求されたページを選択し、それらをグリッドに配置します。ページの各グループが1つの出力ページになります。

たとえば、レイアウトが2列2行に設定されている場合、このツールは各出力ページに最大4つの元ページを配置します。

ページは左から右へ、上から下へ配置されます。

## ページの選択

ページ範囲を使用して、含めるページを選択できます。

ページ範囲を指定しない場合、ツールはPDF内のすべてのページを使用します。

例:

| ページ範囲     | 意味                      |
| --------- | ----------------------- |
| `1-4`     | 1ページ目から4ページ目までを使用する     |
| `1,3,5-8` | 1、3、5ページ目から8ページ目までを使用する |
| `2-`      | 2ページ目から最後のページまでを使用する    |
| `-10`     | 1ページ目から10ページ目までを使用する    |
| 空欄        | すべてのページを使用する            |

ページ番号は通常の1始まりの形式で書かれるため、ページ`1`はPDFの最初のページを意味します。

## 基本的な使い方

入力PDFと出力PDFを指定して、コマンドラインからツールを実行します。

```bash
python nup_pdf.py input.pdf output.pdf 
```

デフォルトでは、すべてのページを使用して2 × 2レイアウトを作成します。

## 行と列の選択

`--cols`と`--rows`を使用してレイアウトを変更できます。

たとえば、これは3 × 2レイアウトを作成します。

```bash
python nup_pdf.py input.pdf output.pdf --cols 3 --rows 2 
```

これにより、各出力ページに最大6つの元ページが配置されます。

## ページ範囲の使用

選択したページのみを使用するには、`--range`を追加します。

```bash
python nup_pdf.py input.pdf output.pdf --range "1-4" 
```

これにより、1ページ目から4ページ目のみが使用されます。

個別のページと範囲を組み合わせることもできます。

```bash
python nup_pdf.py input.pdf output.pdf --range "1,3,5-8" 
```

## 元のページサイズを維持する

デフォルトでは、このツールは元ページを縮小しません。代わりに、グリッド全体を収めるのに十分な大きさの出力ページを作成します。

たとえば、4つのA4ページを2 × 2レイアウトに配置すると、出力ページはおおよそA2サイズになります。

これは、元のページサイズを保持し、テキストや画像の品質低下を避けたい場合に便利です。

## ページを収まるように拡大縮小する

出力ページを入力PDFの最初のページと同じサイズに保ちたい場合は、`--scale-to-fit`を使用します。

```bash
python nup_pdf.py input.pdf output.pdf --scale-to-fit 
```

このモードでは、各元ページがグリッドセル内に収まるように縮小されます。

これは、複数のページを1枚のシートに配置する、通常のプリンター形式のn-up出力に近いものです。

## 余白と間隔の追加

`--margin`で外側の余白を、`--gap`でグリッドセル間の間隔を追加できます。

```bash
python nup_pdf.py input.pdf output.pdf --margin 20 --gap 10 
```

どちらの値もPDFポイントで測定されます。

1ポイントは1/72インチです。

余白は出力ページの外側の周囲に追加されます。間隔は配置されたページ間に追加されます。

## コマンド例

### すべてのページを使用して2 × 2 PDFを作成する

```bash
python nup_pdf.py input.pdf output.pdf 
```

### 1ページ目から8ページ目のみを使用して2 × 2 PDFを作成する

```bash
python nup_pdf.py input.pdf output.pdf --range "1-8" 
```

### 3 × 2レイアウトを作成する

```bash
python nup_pdf.py input.pdf output.pdf --cols 3 --rows 2 
```

### プリンター形式のコンパクト版を作成する

```bash
python nup_pdf.py input.pdf output.pdf --scale-to-fit 
```

### 余白と間隔を追加する

```bash
python nup_pdf.py input.pdf output.pdf --margin 24 --gap 12 
```

## このツールを使用する場面

このツールは、次のような場合に役立ちます。

* スライドPDFから配布資料を作成する
* 複数の文書ページを1つの概要ページにまとめる
* 印刷枚数を削減する
* コンパクトな復習資料を作成する
* 選択したPDFページを整ったグリッドに配置する

## まとめ

このPDFページ結合ツールは、既存のPDFから選択したページをグリッド状に配置して新しいPDFを作成します。柔軟なページ選択、カスタムの行と列、任意の拡大縮小、余白、間隔に対応しています。

元のページサイズを保持したい場合はデフォルトモードを使用してください。複数ページを元のサイズのページに収めたい場合は、`--scale-to-fit`を使用してください。

```python
from __future__ import annotations

from pathlib import Path

from pypdf import PdfReader, PdfWriter, PageObject, Transformation


def parse_page_range(page_range: str | None, total_pages: int) -> list[int]:
    """
    Convert a 1-based page range string to a list of 0-based page indices.

    If page_range is None or empty, all pages are selected.

    Examples
    --------
    "1-4"     -> [0, 1, 2, 3]
    "1,3,5-7" -> [0, 2, 4, 5, 6]
    "2-"      -> from page 2 to the last page
    "-5"      -> from page 1 to page 5
    "" or None -> all pages
    """
    if page_range is None or not page_range.strip():
        return list(range(total_pages))

    pages: list[int] = []

    for part in page_range.split(","):
        part = part.strip()
        if not part:
            continue

        if "-" in part:
            start_s, end_s = part.split("-", 1)

            start = int(start_s) if start_s else 1
            end = int(end_s) if end_s else total_pages

            if start < 1 or end > total_pages or start > end:
                raise ValueError(f"Invalid page range: {part}")

            pages.extend(range(start - 1, end))
        else:
            p = int(part)
            if p < 1 or p > total_pages:
                raise ValueError(f"Page does not exist: {p}")
            pages.append(p - 1)

    if not pages:
        return list(range(total_pages))

    return pages


def nup_pdf(
    input_pdf: str | Path,
    output_pdf: str | Path,
    page_range: str | None = None,
    cols: int = 2,
    rows: int = 2,
    *,
    scale_to_fit: bool = False,
    margin: float = 0,
    gap: float = 0,
) -> None:
    """
    Combine selected PDF pages into a cols x rows grid on each output page.

    Parameters
    ----------
    input_pdf:
        Input PDF file.
    output_pdf:
        Output PDF file.
    page_range:
        Page range to use, expressed with 1-based page numbers.
        If omitted, all pages are used.
        Examples: "1-4", "1,3,5-8", "2-", "-10"
    cols:
        Number of pages in the horizontal direction. Use 2 for a 2x2 layout.
    rows:
        Number of pages in the vertical direction. Use 2 for a 2x2 layout.
    scale_to_fit:
        False:
            Preserve the original page size and create a larger output page.
            For example, a 2x2 layout of A4 pages is roughly A2-sized.
        True:
            Keep the output page size equal to the first input page and shrink
            each source page into its grid cell. This is closer to ordinary
            n-up printing behavior.
    margin:
        Outer margin of the output page, in PDF points.
        1 point = 1/72 inch.
    gap:
        Gap between cells, in PDF points.
    """
    if cols <= 0 or rows <= 0:
        raise ValueError("cols and rows must be 1 or greater.")

    reader = PdfReader(str(input_pdf))
    writer = PdfWriter()

    selected_indices = parse_page_range(page_range, len(reader.pages))
    if not selected_indices:
        raise ValueError("No pages were selected.")

    pages_per_sheet = cols * rows

    # Reference page size.
    first_page = reader.pages[selected_indices[0]]
    base_width = float(first_page.mediabox.width)
    base_height = float(first_page.mediabox.height)

    if scale_to_fit:
        # Keep the output size equal to one original PDF page.
        out_width = base_width
        out_height = base_height

        cell_width = (out_width - 2 * margin - gap * (cols - 1)) / cols
        cell_height = (out_height - 2 * margin - gap * (rows - 1)) / rows
    else:
        # Do not shrink pages; create a larger page for the full grid.
        cell_width = base_width
        cell_height = base_height

        out_width = cols * cell_width + gap * (cols - 1) + 2 * margin
        out_height = rows * cell_height + gap * (rows - 1) + 2 * margin

    for group_start in range(0, len(selected_indices), pages_per_sheet):
        group = selected_indices[group_start : group_start + pages_per_sheet]

        output_page = PageObject.create_blank_page(
            width=out_width,
            height=out_height,
        )

        for slot, page_index in enumerate(group):
            src_page = reader.pages[page_index]

            src_width = float(src_page.mediabox.width)
            src_height = float(src_page.mediabox.height)

            col = slot % cols
            row = slot // cols

            # The PDF coordinate system starts at the lower-left corner.
            # Flip the y-position calculation so row=0 is placed on the top row.
            x0 = margin + col * (cell_width + gap)
            y0 = margin + (rows - 1 - row) * (cell_height + gap)

            if scale_to_fit:
                scale = min(cell_width / src_width, cell_height / src_height)
            else:
                scale = 1.0

            placed_width = src_width * scale
            placed_height = src_height * scale

            # Center the source page within the cell.
            tx = x0 + (cell_width - placed_width) / 2
            ty = y0 + (cell_height - placed_height) / 2

            # Also support PDFs whose mediabox lower-left corner is not (0, 0).
            src_left = float(src_page.mediabox.left)
            src_bottom = float(src_page.mediabox.bottom)

            transform = (
                Transformation()
                .translate(tx=-src_left, ty=-src_bottom)
                .scale(scale, scale)
                .translate(tx=tx, ty=ty)
            )

            output_page.merge_transformed_page(
                src_page,
                transform,
                over=True,
            )

        writer.add_page(output_page)

    with open(output_pdf, "wb") as f:
        writer.write(f)


import argparse


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Combine PDF pages into an n-up grid layout."
    )
    parser.add_argument("input_pdf")
    parser.add_argument("output_pdf")
    parser.add_argument(
        "--range",
        dest="page_range",
        default=None,
        help="1-based page range to use, such as '1-4', '1,3,5-8', '2-', or '-10'. If omitted, all pages are used.",
    )
    parser.add_argument("--cols", type=int, default=2)
    parser.add_argument("--rows", type=int, default=2)
    parser.add_argument("--scale-to-fit", action="store_true")
    parser.add_argument("--margin", type=float, default=0)
    parser.add_argument("--gap", type=float, default=0)

    args = parser.parse_args()

    nup_pdf(
        input_pdf=args.input_pdf,
        output_pdf=args.output_pdf,
        page_range=args.page_range,
        cols=args.cols,
        rows=args.rows,
        scale_to_fit=args.scale_to_fit,
        margin=args.margin,
        gap=args.gap,
    )


if __name__ == "__main__":
    main()
```
