---
pubDatetime: 2026-06-26T21:20:40+09:00
title: "GitHub Releases の表示順"
description: "結論 GitHub Releases の表示順は、タイトル順ではありません。また、タグ名の単純な文字列順でもありません。公開日時 の単純な降順でもありません。 GitHub Releases の表示順をコントロールするためには、タグ名を SemVer形式にすることをすすめます。 GitHub はリリ…"
---

## 結論

GitHub Releases の表示順は、タイトル順ではありません。また、タグ名の単純な文字列順でもありません。公開日時 の単純な降順でもありません。

GitHub Releases の表示順をコントロールするためには、**タグ名を SemVer形式**にすることをすすめます。

GitHub はリリースを次のような基準の組み合わせで扱っています。

| 観点                 | 使われるもの                                                           |
| ------------------ | ---------------------------------------------------------------- |
| 「Latest」判定         | 明示的な `Set as latest release` / `make_latest`、または SemVer ベースの自動判定 |
| `/releases/latest` | draft / prerelease を除いた「latest」扱いのリリース                           |
| Releases 一覧の見た目    | 公開日時ではなく、タグ・コミット日時や SemVer 的な扱いが絡む。完全なソート仕様は明文化されていない            |

GitHub の UI では、リリース作成時に **Set as latest release** を選べます。これを選ばない場合、latest ラベルは semantic versioning に基づいて自動割り当てされる、と公式 Docs に書かれています。([GitHub Docs][1])

## 日時順に見えない理由

GitHub REST API の `Get the latest release` の説明では、latest は `created_at` で決まるとされ、その `created_at` は「リリースを draft / publish した日時」ではなく、**リリースに使われたコミットの日付**だと説明されています。([GitHub Docs][2])

つまり、たとえば次のようなことが起きます。

```text
2026-06-26 に Release A を公開
  -> ただしタグは 2025 年の古いコミットを指している

2026-06-20 に Release B を公開
  -> タグは 2026 年の新しいコミットを指している
```

この場合、見た目の公開日時では A の方が新しくても、GitHub 側の判定では B の方が「新しい」扱いになる可能性があります。

## タグ名順か？

**単純なタグ名順ではありません。**
ただし、タグ名が SemVer として解釈できる場合は、GitHub の判定に影響します。

GitHub の公式 Changelog でも、以前は「最も新しい日付のリリース」が latest で、同じ日付の場合は semantic version number がタイブレークに使われていた、と説明されています。現在は明示的に latest を指定できる機能が追加されています。([The GitHub Blog][3])

また、GitHub Community の議論では、GitHub 側の説明として、同日のリリースは SemVer で並べる想定で、基本的には GitHub release 自体の日時ではなく **tag / commit の日付**に基づく、という説明があります。([GitHub][4])

そのため、次のようなタグは並びが崩れやすいです。

```text
release/69.3
build-20240626
2024-06-26-001
v1.0
v1.0.0-beta.10
v1.0.0-beta.9
```

SemVer として扱わせたいなら、基本は次の形式が無難です。

```text
v1.2.3
v1.2.4
v1.3.0
v2.0.0
```

## タイトル順か？

**タイトル順ではありません。**

Release title、つまり `name` は表示用です。GitHub API でも `tag_name` と `name` は別フィールドとして扱われています。ソートや latest 判定で主に問題になるのは、タイトルではなく、タグ、SemVer、`created_at`、`published_at`、`make_latest` です。([GitHub Docs][5])

## API で取得する場合の注意

`GET /repos/{owner}/{repo}/releases`、つまり List releases は、公式 REST Docs 上では `per_page` と `page` はありますが、`sort` や `direction` のような明示的なソート指定はありません。([GitHub Docs][5])
GitHub Community の過去回答でも、API の返却順に特定の保証はないので依存しない方がよい、という趣旨の説明があります。([GitHub][6])

したがって、プログラムで正確に扱うなら、自分でソートするのが安全です。

```bash
gh release list \
  --repo OWNER/REPO \
  --limit 100 \
  --json tagName,name,publishedAt,createdAt,isLatest,isPrerelease,isDraft \
  --jq '.[] | [.tagName, .name, .publishedAt, .createdAt, .isLatest, .isPrerelease, .isDraft] | @tsv'
```

`gh release list` は `createdAt`, `publishedAt`, `tagName`, `name`, `isLatest`, `isPrerelease`, `isDraft` などを JSON で出せます。([GitHub CLI][7])

## 実務上の整理

見た目の順番を安定させたいなら、次の運用が堅いです。

1. タグは SemVer 形式にする
   `v1.2.3` のようにする。

2. 古いコミットに対して後から Release を作ると、表示順が期待とずれる可能性がある
   公開日時ではなく、タグ・コミット側の日付が効くためです。

3. 「最新」として見せたいリリースは明示的に latest にする
   UI なら **Set as latest release**、API なら `make_latest: true` を使います。`make_latest` は `true`, `false`, `legacy` を指定でき、draft / prerelease は latest にできません。([GitHub Docs][2])

4. API やスクリプトでは GitHub の返却順に依存しない
   `publishedAt` 順、`createdAt` 順、SemVer 順など、目的に応じて自前で並べるのが安全です。

まとめると、**GitHub Releases の順番は「タイトル順」でも「タグ名の単純順」でも「公開日時順」でもなく、GitHub 内部の latest / SemVer / タグ・コミット日時系の判定が絡む**、という理解でよいです。

[1]: https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository "Managing releases in a repository - GitHub Docs"
[2]: https://docs.github.com/rest/releases/releases "REST API endpoints for releases - GitHub Docs"
[3]: https://github.blog/changelog/2022-10-20-explicitly-set-the-latest-release/ "Explicitly Set the Latest Release - GitHub Changelog"
[4]: https://github.com/orgs/community/discussions/8226 "Releases out of order · community · Discussion #8226 · GitHub"
[5]: https://docs.github.com/ja/rest/releases/releases "リリースの REST API エンドポイント - GitHubドキュメント"
[6]: https://github.com/orgs/community/discussions/21901 "Releases API - can't figure out order · community · Discussion #21901 · GitHub"
[7]: https://cli.github.com/manual/gh_release_list?utm_source=chatgpt.com "gh release list"
