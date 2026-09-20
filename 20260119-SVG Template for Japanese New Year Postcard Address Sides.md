---
pubDatetime: 2026-01-19T17:33:05+09:00
title: "年賀状はがき宛名面用SVGテンプレート"
description: "表示上の注意（iOS Safari）、今後の展望について、具体的な手順と注意点をまとめます。"
---

年賀状の宛名印刷を効率化することを目的として、**はがき宛名面のテンプレートとなるSVGファイル**を作成した。

本テンプレートは、SVG内に記述された住所や氏名等のテキストを置換し、**Inkscapeを用いてPDF形式に変換した上で印刷**することを前提としている。テキスト情報を直接編集できるため、レイアウトの調整や自動化が比較的容易である点が特徴である。

なお、テンプレート内のテキスト要素は**下寄せ（ベースライン揃え）** および **スペーシング（文字間・行間）** をあらかじめ設定している。これにより、住所や氏名などの **文字列長が変動してもレイアウトが崩れにくく、常に意図した位置に適切に配置される** ようにしている。

## 表示上の注意（iOS Safari）

iOS版SafariでSVGを表示した場合、**長音記号「ー」** が意図した字形ではなく、**単なる横棒のように表示される場合がある**ことを確認している。  
これは使用フォントやブラウザ側のレンダリング仕様に起因するものであり、Safari固有の挙動である可能性が高い。

そのため、

* iOS Safari 上での表示確認は参考程度に留める  
* 最終的なレイアウト確認は **Inkscape上**、もしくは **PDF変換後** に行う  

ことを推奨する。

## 今後の展望

現時点ではSVGテンプレートのみの提供であるが、今後は以下のような機能拡張を検討している。

* 住所録データ（CSV等）の読み込み
* 宛名情報の自動差し替え
* 複数宛先分のPDF一括生成

年賀状作成に伴う作業負荷を軽減することを目的に、引き続き改善を進めていく予定である。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg width="10cm" height="14.8cm" version="1.1" viewBox="0 0 283.465 419.528" xmlns="http://www.w3.org/2000/svg">
<text x="19.362963" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="19.362963" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">1</tspan></text>
<text x="30.84087" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="30.84087" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">2</tspan></text>
<text x="40.913643" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="40.913643" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">3</tspan></text>
<text x="55.981144" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="55.981144" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">4</tspan></text>
<text x="66.940186" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="66.940186" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">5</tspan></text>
<text x="78.733505" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="78.733505" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">6</tspan></text>
<text x="90.001259" y="366.50623" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="14.1437px" stroke-width=".883983" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="90.001259" y="366.50623" stroke-width=".883983" text-align="center" text-anchor="middle">7</tspan></text>
<text x="72.195801" y="184.05423" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="9.81849px" stroke-width=".565399" writing-mode="tb-rl" xml:space="preserve"><tspan x="72.195801" y="184.05423" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="9.81849px" stroke-width=".565399" writing-mode="tb-rl">ほげほげ県ほげほげ市</tspan></text>
<text x="60.137417" y="342.7861" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="9.81849px" stroke-width=".613652" text-anchor="end" writing-mode="tb-rl" xml:space="preserve"><tspan x="60.137417" y="342.7861" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width=".613652" text-anchor="end" writing-mode="tb-rl">ほげ区ほげ町一ー二ー三</tspan></text>
<text x="46.161102" y="286.2576" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="13.1262px" stroke-width=".820383" text-anchor="middle" writing-mode="tb-rl" textLength="121.14465" xml:space="preserve"><tspan x="46.161102" y="286.2576" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width=".820383" text-anchor="middle" writing-mode="tb-rl">ほげ ほげ男</tspan></text>
<text x="133.54039" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="133.54039" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">1</tspan></text>
<text x="153.46494" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="153.46494" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">2</tspan></text>
<text x="173.44075" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="173.44075" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">3</tspan></text>
<text x="194.99701" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="194.99701" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">4</tspan></text>
<text x="214.03473" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="214.03473" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">5</tspan></text>
<text x="233.40883" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="233.40883" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">6</tspan></text>
<text x="252.96613" y="55.380344" font-family="Yu Gothic, MS PGothic, sans-serif" font-size="21.1158px" stroke-width="1.31974" xml:space="preserve" text-align="center" text-anchor="middle"><tspan x="252.96613" y="55.380344" stroke-width="1.31974" text-align="center" text-anchor="middle">7</tspan></text>
<text x="246.77936" y="87.852119" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="16.0466px" stroke-width="1.00291" writing-mode="tb-rl" xml:space="preserve"><tspan x="246.77936" y="87.852119" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width="1.00291" writing-mode="tb-rl">ほげほげ県ほげほげ市</tspan></text>
<text x="225.63652" y="347.62384" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="16.0466px" stroke-width="1.00291" text-anchor="end" writing-mode="tb-rl" xml:space="preserve"><tspan x="225.63652" y="347.62384" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width="1.00291" text-anchor="end" writing-mode="tb-rl">ほげほげほげ区ほげ町1-2-3</tspan></text>
<text x="204.46429" y="87.8423" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="16.0466px" stroke-width="1.00291" writing-mode="tb-rl" xml:space="preserve"><tspan x="204.46429" y="87.8423" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width="1.00291" writing-mode="tb-rl">ほげほげほげ会社ほげほげ本部</tspan></text>
<text x="181.51776" y="347.94901" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="16.0466px" stroke-width="1.00291" text-anchor="end" writing-mode="tb-rl" xml:space="preserve"><tspan x="181.51776" y="347.94901" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width="1.00291" text-anchor="end" writing-mode="tb-rl">ほげほげ部ほげほげ課</tspan></text>
<text x="146.31149" y="239.40382" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" font-size="27.3034px" stroke-width="1.70646" text-anchor="middle" writing-mode="tb-rl" textLength="224.07112" xml:space="preserve"><tspan x="146.31149" y="239.40382" font-family="HGPKyokashotai, Yu Mincho, MS PMincho, serif" stroke-width="1.70646" text-anchor="middle" writing-mode="tb-rl">ほげ ほげ子 様</tspan></text>
</svg>
```
