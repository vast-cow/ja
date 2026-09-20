---
title: "Git Rebaseで過去のコミットをSquashする方法"
description: ""
pubDatetime: 2026-08-24T08:20:38.714Z
---

Gitの`rebase`コマンドを使ってコミットをSquash（統合）すると、コミット履歴を整理したいときに便利です。以下に手順を説明します。

---

## 1. Squashするコミットを確認する

まず、コミットログを確認して、どのコミットをまとめるか決めます。

```bash
git log --oneline
```

例：

```plaintext
a1b2c3d タイポを修正
d4e5f6g 機能Xを追加
h7i8j9k WIP: 機能Xを実装
...
```

ここでは、`機能Xを追加`と`WIP: 機能Xを実装`をSquashするとします。

---

## 2. Interactive Rebaseを開始する

Squashしたいコミットの**直前**のコミットをベースとして、`git rebase -i`を実行します。

```bash
git rebase -i <base_commit_id>
```

この例で、`a1b2c3d`がその直前のコミットである場合は、次のように実行します。

```bash
git rebase -i a1b2c3d
```

---

## 3. コミット一覧を編集する

エディタが開き、次のような内容が表示されます。

```plaintext
pick d4e5f6g 機能Xを追加
pick h7i8j9k WIP: 機能Xを実装
```

2つ目の`pick`を`squash`（または単に`s`）に変更します。

```plaintext
pick d4e5f6g 機能Xを追加
squash h7i8j9k WIP: 機能Xを実装
```

---

## 4. コミットメッセージを編集する

次に、Gitからコミットメッセージを統合するよう求められます。

```plaintext
# これは2個のコミットを組み合わせたものです。
# 1つ目のコミットメッセージ:
機能Xを追加

# 以下のコミットメッセージも含まれます:
WIP: 機能Xを実装
```

最終的なメッセージが簡潔になるよう整理します。

```plaintext
機能Xを追加
```

---

## 5. Rebaseを完了する

保存してエディタを閉じます。GitがRebaseを適用し、コミットを1つにSquashします。

---

## 6. トラブルシューティング

* **コンフリクトが発生した場合**：通常どおりコンフリクトを解消してから、次を実行します。

  ```bash
  git add <fixed_files>
  git rebase --continue
  ```

* **Rebaseをキャンセルしたい場合**：

  ```bash
  git rebase --abort
  ```

* **履歴を書き換えた後**：ブランチをすでにリモートへPushしている場合は、書き換えた履歴をForce Pushする必要があります。

  ```bash
  git push --force-with-lease
  ```

  （共同作業をしている場合は、`--force`より`--force-with-lease`を使用することを推奨します。）

---

## 7. ルートコミットを含めてSquashする（`--root`）

デフォルトでは、`git rebase -i <base>`は指定したベースコミットの**次**のコミットからRebaseを行います。最初のコミット（ルートコミット）もInteractive Rebaseの対象に含めたい場合（たとえば、初回コミットを編集したり、後続のコミットとSquashしたりする場合）は、`--root`オプションを使用します。

```bash
git rebase -i --root
```

**このコマンドの動作**

* ルート（初回）コミットがInteractive Rebaseのtodoリストの最初の項目として表示されます。
* 他のコミットと同様に、ルートコミットに対して`pick`、`reword`、`edit`、`squash`、`fixup`を指定できます。
* ルートコミットと、それ以降のすべてのコミットが書き換えられます（そのため、ブランチ内のすべてのコミットIDが変更されます）。

**一般的な用途と例**

1. *すべてのコミットを1つの初回コミットにSquashする*
   履歴全体を含む1つのコミットにしたい場合は、ルートコミットを`pick`のままにして、それ以外のすべてのコミットを`squash`（または`s`）に変更します。

   ```plaintext
   pick a1b2c3d 初回コミット
   squash d4e5f6g 機能Xを追加
   squash h7i8j9k WIP: 機能Xを実装
   ...
   ```

   保存すると、Gitが統合後のコミットメッセージを編集するためのエディタを開くので、最終的なメッセージを作成できます。

2. *ルートコミットのメッセージを編集・変更する*
   Rebase中に初回コミットのメッセージを変更するには、最初の行を`reword`（または`r`）に変更します。

   ```plaintext
   reword a1b2c3d 初回コミット
   pick d4e5f6g 機能Xを追加
   ...
   ```

   Rebaseが一時停止し、ルートコミットのメッセージを編集するよう求められます。

3. *ルートコミットの内容を修正する*
   ルートコミットの実際のツリー（ファイルの追加・削除）を変更する必要がある場合は、ルートのエントリに`edit`を指定します。Rebaseはそのコミットで一時停止します。その後、ファイルを変更して`git add`し、`git commit --amend`を実行してからRebaseを続行できます。

   ```bash
   # Interactive Rebaseのtodo:
   edit a1b2c3d 初回コミット
   pick d4e5f6g 機能Xを追加
   ...
   # Rebaseが停止したら:
   # ワーキングツリーに変更を加える
   git add <files>
   git commit --amend --no-edit   # 必要に応じてメッセージも編集
   git rebase --continue
   ```

**注意点**

* ルートコミットを書き換えると、ブランチの履歴全体が書き換えられます。共有ブランチで行うと大きな影響があるため、共同作業者と調整するか、公開済みブランチの書き換えは避けてください。
* `--root`を使ってRebaseした後、リモートブランチを更新するにはForce Pushが必要です（`git push --force-with-lease`）。
* `git rebase -i --root`でも、コンフリクトの解消や中止（`git rebase --abort`）については通常のInteractive Rebaseと同様に動作します。

---

## まとめ

* `git rebase -i <base>`を使用すると、`<base>`より後のコミットを対話形式でSquashできます。
* Squashするコミットを`squash`（または`s`）に変更します。
* プロンプトが表示されたらコミットメッセージを編集します。
* Rebaseを完了し、コンフリクトが発生した場合は`git rebase --continue`で続行します。
* 初回／ルートコミットをRebaseの対象に含めてSquashや編集を行うには、`git rebase -i --root`を実行します。この操作ではブランチの履歴全体が書き換えられ、公開済みブランチではForce Pushが必要になる点に注意してください。
