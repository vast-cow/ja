---
title: "`git commit --amend` でコミット日時を維持する方法"
description: "環境変数 GIT_COMMITTER_DATE に元のコミット日時を指定して --amend を実行する方法を紹介。"
pubDatetime: 2026-11-25T22:58:00+09:00
---

`git commit --amend` でコミット日時を変更したくない場合は、`GIT_COMMITTER_DATE` に変更前の日時を指定します。

```
GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend
```

コミットメッセージも変更しない場合：

```
GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend --no-edit
```

## **Gitの日時の違い**

| 日時                 | 通常の `--amend` | 上記のコマンド |
| ------------------ | ------------- | ------- |
| AuthorDate（作成日時）   | 維持            | 維持      |
| CommitDate（コミット日時） | 更新            | 維持      |

以下のコマンドで両方の日時を確認できます。

```
git log -1 --format=fuller
```

## **エイリアスに登録する（おすすめ）**

頻繁に使う場合は、Gitエイリアスに登録しておくと便利です。

```
git config --global alias.amend-keep-date '!f() { GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend "$@"; }; f'
```

以降は以下のコマンドだけで実行できます。

```
git amend-keep-date
```

```
git amend-keep-date --no-edit
```

**注意：** 日時を維持しても、コミット内容やメッセージを変更すればコミットハッシュは変わります。
