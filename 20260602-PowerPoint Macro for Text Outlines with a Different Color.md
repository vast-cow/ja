---
pubDatetime: 2026-06-02T21:52:15+09:00
title: "PowerPointで文字色と異なる輪郭をつけた文字を作るマクロ"
description: "PowerPointで、文字の中身の色を保ったまま、別の色の輪郭をつけたいことがあります。通常の方法で文字に輪郭を設定すると、輪郭が文字の内側にも入り込み、文字色がつぶれて見えることがあります。 このマクロは、その問題を避けるために、輪郭用のテキストボックスと中身用のテキストボックスを重ねて使う仕組…"
---

PowerPointで、文字の中身の色を保ったまま、別の色の輪郭をつけたいことがあります。通常の方法で文字に輪郭を設定すると、輪郭が文字の内側にも入り込み、文字色がつぶれて見えることがあります。

このマクロは、その問題を避けるために、**輪郭用のテキストボックス**と**中身用のテキストボックス**を重ねて使う仕組みです。

## このマクロの目的

このマクロの目的は、PowerPoint上で次のような文字を簡単に作ることです。

* 文字の中身は元の文字色のままにする
* 文字の外側に、別の色の太い輪郭をつける
* 輪郭によって文字の中身がつぶれないようにする

たとえば、背景に写真や濃い色の図形があるスライドで、文字を読みやすくしたい場合に便利です。

## 仕組み

このマクロでは、選択したテキストボックスを複製します。

元のテキストボックスには輪郭を設定し、複製したテキストボックスは文字の中身用として重ねます。最後に、この2つのテキストボックスをグループ化します。

通常の文字輪郭だけでは文字の中身が狭く見えたり、色がつぶれたりすることがありますが、2つのテキストボックスを重ねることで、見た目をきれいに保ちやすくなります。

## 使い方

### 1. テキストボックスを作成する

まず、PowerPoint上にテキストボックスを作成します。

そのテキストボックスには、表示したい文字を入力しておきます。

### 2. テキストボックスに背景色を設定する

次に、テキストボックスに背景色を設定します。

この背景色が、文字の輪郭色として使われます。たとえば、白い輪郭をつけたい場合は、テキストボックスの背景色を白にします。

### 3. テキストボックスを1つだけ選択する

マクロを実行する前に、対象のテキストボックスを1つだけ選択します。

図形や複数のオブジェクトを選択している場合、このマクロは実行されません。

### 4. マクロを実行する

テキストボックスを選択した状態で、マクロを実行します。

実行すると、選択したテキストボックスの背景色が取得され、その色で文字に太さ6ptの輪郭が設定されます。その後、同じ位置に複製されたテキストボックスと重ねられ、2つのテキストボックスがグループ化されます。

## マクロ本体

```vb
Sub TextBoxOutlineFromBackground()

    Dim shpA As Shape
    Dim shpC As Shape
    Dim fillColor As Long
    Dim sr As ShapeRange
    Dim sld As Slide

    ' 現在選択中のオブジェクトを取得
    If ActiveWindow.Selection.Type <> ppSelectionShapes Then
        MsgBox "図形を1つ選択してください。", vbExclamation
        Exit Sub
    End If

    If ActiveWindow.Selection.ShapeRange.Count <> 1 Then
        MsgBox "オブジェクトは1つだけ選択してください。", vbExclamation
        Exit Sub
    End If

    Set shpA = ActiveWindow.Selection.ShapeRange(1)

    ' オブジェクトAがテキストボックスであることを確認
    If shpA.Type <> msoTextBox Then
        MsgBox "選択中のオブジェクトはテキストボックスではありません。", vbExclamation
        Exit Sub
    End If

    If Not shpA.HasTextFrame Then
        MsgBox "選択中のオブジェクトにはテキストフレームがありません。", vbExclamation
        Exit Sub
    End If

    If Not shpA.TextFrame.HasText Then
        MsgBox "選択中のテキストボックスに文字がありません。", vbExclamation
        Exit Sub
    End If

    ' オブジェクトAに背景色があることを確認
    If shpA.Fill.Visible <> msoTrue Then
        MsgBox "選択中のテキストボックスには背景色がありません。", vbExclamation
        Exit Sub
    End If

    ' 背景色を取得して色Bとする
    fillColor = shpA.Fill.ForeColor.RGB

    shpA.Fill.Visible = msoFalse

    ' 複製してCを作る
    Set shpC = shpA.Duplicate(1)

    shpC.Left = shpA.Left
    shpC.Top = shpA.Top
    shpC.Width = shpA.Width
    shpC.Height = shpA.Height

    ' Aの背景色を無しにする
    shpA.Fill.Visible = msoFalse

    ' Aの文字に背景色と同じ色で6ptの輪郭を設定
    With shpA.TextFrame2.TextRange.Font.Line
        .Visible = msoTrue
        .ForeColor.RGB = fillColor
        .Weight = 6
        .Transparency = 0
    End With

    ' 選択中Shapeが存在するスライド上でAとCをグループ化
    Set sld = shpA.Parent
    Set sr = sld.Shapes.Range(Array(shpA.Name, shpC.Name))
    sr.Group.Select

End Sub
```

## クイックアクセスやリボンに登録する

このマクロは、PowerPointのクイックアクセスツールバーやリボンに登録しておくと便利です。

よく使う場合は、毎回マクロ一覧から実行するよりも、ボタンとして登録しておくとすぐに呼び出せます。

クイックアクセスツールバーやリボンへの登録方法は、以下の記事が参考になります。

https://qiita.com/FollowUser/items/25604896eb78c8d3a053

## 使用時の注意点

このマクロを使うには、事前にテキストボックスへ背景色を設定しておく必要があります。背景色が設定されていない場合、輪郭色を決められないため、マクロは実行されません。

また、選択できるオブジェクトはテキストボックス1つだけです。複数のオブジェクトを選択している場合や、通常の図形を選択している場合は、エラーメッセージが表示されます。

## まとめ

このマクロを使うと、PowerPointで文字色を保ったまま、別の色の輪郭をつけた文字を作れます。

輪郭用と中身用のテキストボックスを重ねることで、文字の中身がつぶれにくくなり、スライド上でも読みやすい文字を作成できます。写真の上に文字を置く場合や、強調した見出しを作る場合に便利です。
