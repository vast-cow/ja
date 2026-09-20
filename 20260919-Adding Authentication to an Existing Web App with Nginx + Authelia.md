---
title: "nginx + Autheliaによる既存Webアプリへの認証機能追加"
description: ""
pubDatetime: 2026-09-19T11:51:27.735Z
---

はい。この条件なら、**「認証は nginx + 外部認証サービスに完全に寄せ、アプリは認証済みユーザー名をヘッダで受け取るだけ」**にするのが適切です。

## 推奨構成

第一候補は **nginx + Authelia** です。

```text
                   Internet
                      │
                 HTTPS :443
                      │
                ┌─────▼─────┐
                │   nginx   │
                │ TLS終端   │
                │ auth_request
                └─────┬─────┘
                      │
             認証確認 │        未認証
        ┌─────────────┘           │
        ▼                         ▼
 ┌─────────────┐           ┌──────────────┐
 │  Authelia   │◀─────────▶│ auth.example │
 │ password/MFA│           │ ログイン画面 │
 └─────────────┘           └──────────────┘
        │
   認証OK
        │
        ▼
 ┌───────────────────┐
 │ nginx             │
 │ Remote-User: foo  │
 │ を強制的に付与     │
 └─────────┬─────────┘
           │ private network only
           ▼
 ┌───────────────────┐
 │ Existing App      │
 │ user = Remote-User│
 └───────────────────┘
```

Authelia は nginx の `auth_request` を正式にサポートしており、認証成功後に `Remote-User`、`Remote-Groups`、`Remote-Email` 等を nginx に返せます。nginx は `auth_request_set` で値を受け、バックエンドへヘッダとして渡せます。([Authelia][1])

### なぜ Authelia か

今回の要件との対応がかなり素直です。

| 要件            | nginx + Authelia |
| ------------- | ---------------- |
| 既存アプリ変更を最小化   | ◎                |
| ユーザー名+パスワード   | ◎                |
| `Remote-User` | ◎ ネイティブ          |
| nginx のまま使う   | ◎                |
| ローカルユーザーDB    | ◎                |
| LDAP連携        | ◎                |
| MFAを後から追加     | ◎                |
| グループによるアクセス制御 | ◎                |
| brute-force対策 | ◎                |
| SSO化          | ◎                |
| 小規模構成         | ◎                |
| 複数アプリへ拡張      | ◎                |

Authelia の file backend は Argon2id を利用でき、認証失敗回数に応じたユーザー/IP単位の一時BANもあります。([Authelia][2])

---

# アプリ側の変更

アプリ側は極端に言えばこれだけです。

```text
Remote-User: alice
```

を、

```text
現在ログインしているユーザー = alice
```

として扱います。

例えば、

```pseudo
username = request.headers["Remote-User"]

if username is empty:
    return 401

user = find_or_create_user(username)
```

程度です。

ただし、**ここに1つ非常に重要なセキュリティ境界があります。**

アプリ自身は、

> `Remote-User` が付いている → 認証済み

と無条件に信用することになるので、**アプリへ nginx を迂回してアクセスできてはいけません。**

---

# 最重要ポイント：アプリを直接公開しない

例えば Docker なら、

```yaml
app:
  expose:
    - "8080"
```

にはするが、

```yaml
ports:
  - "8080:8080"
```

にはしません。

構成として、

```text
Internet ──► nginx ──► app
                 │
                 └──► Authelia
```

だけを許可します。

以下はNGです。

```text
Internet ──► nginx ──► app
   │
   └────────────────► app:8080   ← NG
```

後者だと攻撃者が直接、

```http
GET /
Remote-User: admin
```

を送れば認証を迂回できるからです。

---

# nginx側も必ずヘッダを上書きする

ブラウザから送られてきた `Remote-User` をそのまま転送してはいけません。

概念的にはこうします。

```nginx
location / {
    auth_request /internal/authelia/authz;

    auth_request_set $authenticated_user $upstream_http_remote_user;

    proxy_set_header Remote-User       $authenticated_user;
    proxy_set_header X-Forwarded-User  $authenticated_user;

    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host  $host;

    proxy_pass http://app:8080;
}
```

ポイントは、

```nginx
proxy_set_header Remote-User $authenticated_user;
```

です。

クライアントが、

```http
Remote-User: admin
```

を送り込んでも、nginx が**認証サービスから取得した値で置換**します。

Authelia の公式 nginx 構成も、

```nginx
auth_request_set $user $upstream_http_remote_user;
proxy_set_header Remote-User $user;
```

という構成になっています。([Authelia][1])

---

# `Remote-User` と `X-Forwarded-User` のどちらを使うか

今回なら **`Remote-User` を推奨**します。

Authelia が直接、

```text
Remote-User
Remote-Groups
Remote-Name
Remote-Email
```

を返すためです。([Authelia][1])

例えば、

```http
Remote-User: tanaka
Remote-Email: tanaka@example.com
Remote-Groups: users,developers
```

のようにできます。

アプリ側では基本的に、

```text
Remote-User
```

だけ見れば十分です。

将来、

```text
admin
editor
viewer
```

などを追加したければ `Remote-Groups` も使えます。

---

# 認証フロー

ユーザーが

```text
https://app.example.com/foo
```

へアクセスすると、

```text
1. Browser → nginx

2. nginx
   └─ auth_request → Authelia

3. Authelia
   ├─ sessionあり → OK
   └─ sessionなし → loginへ

4. ユーザー
   username/password入力

5. Authelia
   └─ session cookie発行

6. 再度 app.example.com/foo

7. nginx → Authelia
            ↓
          alice

8. nginx → App
   Remote-User: alice
```

という動作になります。

nginx の `auth_request` は、認証サブリクエストが 2xx なら許可、401/403 なら拒否する仕組みです。([Nginx][3])

---

# Authelia側

少人数ならユーザー情報をファイルで持てます。

概念として、

```yaml
authentication_backend:
  file:
    path: /config/users.yml
    password:
      algorithm: argon2
      argon2:
        variant: argon2id
```

です。

現在の Authelia の file backend は Argon2id を推奨設定として持っています。([Authelia][2])

例えばユーザーを、

```text
alice
bob
charlie
```

と登録。

パスワードそのものは保存せず、

```text
$argon2id$...
```

というハッシュを保存します。

---

# インターネット公開なら `default deny`

Authelia側は、

```yaml
access_control:
  default_policy: deny

  rules:
    - domain: app.example.com
      policy: one_factor
```

という思想がよいです。

Authelia自身も `default_policy: deny` を推奨しています。([Authelia][4])

つまり設定を間違えても、

```text
設定なし → 公開
```

ではなく、

```text
設定なし → 拒否
```

になります。

---

# パスワードだけで始めてもMFAへ移行可能

最初は、

```yaml
policy: one_factor
```

として、

```text
username
password
```

のみ。

あとから、

```yaml
policy: two_factor
```

へ変更すれば、

```text
password
+
TOTP / WebAuthn等
```

という構成へ持っていけます。

Authelia は `one_factor` / `two_factor` をアクセス制御ポリシーとして持っています。([Authelia][4])

公開インターネットに置くサービスなら、特に管理者だけでも最終的には2FAを使える構成にしておく価値があります。

---

# brute-force対策

単純な nginx Basic Auth と比較して大きい部分です。

Authelia には例えば、

```yaml
regulation:
  modes:
    - user
    - ip
  max_retries: 5
  find_time: 2m
  ban_time: 15m
```

のような認証試行制限があります。

現行ドキュメントでも、username/password endpoint への試行回数に応じてユーザー/IPを一時BANできる仕組みが用意されています。([Authelia][5])

---

# ストレージ構成

## 小規模・単一サーバー

これで十分です。

```text
nginx
Authelia
Existing App
SQLite
users.yml
```

Authelia はローカルSQLiteストレージを利用できます。公式例でもこの構成があります。([Authelia][6])

Docker Composeならかなり簡単になります。

```text
docker compose
├── nginx
├── authelia
└── app
```

* volume:

```text
authelia/config
authelia/db.sqlite3
```

---

## HAが必要になったら

その時点で、

```text
        nginx
          │
   ┌──────┴──────┐
Authelia1    Authelia2
   │              │
   └──────┬───────┘
          │
     PostgreSQL
          +
        Redis
```

へ移行できます。

Authelia のセッションストレージは単一インスタンス向けの memory と、HA向けの Redis / Redis Sentinel をサポートしており、HA用途では stateless な Redis 系が推奨されています。([Authelia][7])

最初からここまでやる必要はないと思います。

---

# TLS

これは必須と考えた方がよいです。

```text
https://app.example.com
https://auth.example.com
```

にします。

例えば、

```text
Let's Encrypt
   ↓
nginx TLS termination
   ↓
internal docker network
```

です。

Authelia自身も、認証ポータルおよび forward authentication で保護するアプリについて HTTPS/WSS を要求しています。([Authelia][8])

---

# nginx Basic Auth というもっと簡単な案

実は「ほぼ一切追加したくない」なら、

```text
nginx
+
.htpasswd
```

だけでもできます。

```nginx
location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/users.htpasswd;

    proxy_set_header Remote-User $remote_user;

    proxy_pass http://app:8080;
}
```

nginx の `$remote_user` には Basic Authentication のユーザー名が入ります。([Nginx][9])

したがって、

```text
Browser
  ↓
Basic Auth
  ↓
nginx
  ↓
Remote-User: alice
  ↓
App
```

という、極めて小さい構成も成立します。

### ただし

インターネット向け一般ユーザーサービスなら、私はこれを**暫定構成**として扱います。

理由は、

* ログイン画面を作れない
* ブラウザのBasic Auth UXに依存
* 明示的なログアウトが扱いづらい
* パスワード変更UIがない
* アカウント管理が `htpasswd`
* MFAがない
* ロックアウト/認証監査が弱い
* 将来的なSSOが難しい

からです。

「管理者3人だけが使うツール」程度なら合理的ですが、一般ユーザーが存在するなら Authelia の方が扱いやすいです。

---

# authentikという選択肢

もう一つ有力なのは、

```text
nginx + authentik
```

です。

authentik にも nginx 用の **Forward Auth** があり、既存アプリ自身を authentik 経由でプロキシする必要はなく、

```text
nginx → authentikへ認証問い合わせ
nginx → existing app
```

という構成ができます。([authentik][10])

認証済みユーザーについて、

```text
X-authentik-username
X-authentik-groups
X-authentik-email
X-authentik-name
X-authentik-uid
```

などを出せます。([authentik][11])

さらに custom header で、

```text
Remote-User
```

へ変換することも可能です。([authentik][12])

ただし今回の用途だけなら、

```text
authentik
```

は Authelia よりやや大掛かりです。

---

# 3案を整理すると

| 構成                   |   手軽さ |  公開向け | ユーザー管理 | MFA | 向くケース     |
| -------------------- | ----: | ----: | -----: | --: | --------- |
| nginx Basic Auth     |     ◎ |     △ |      △ |   × | 数人用       |
| **nginx + Authelia** | **○** | **◎** |      ○ |   ◎ | **今回**    |
| nginx + authentik    |     △ |     ◎ |      ◎ |   ◎ | 本格IAM/SSO |

今回の条件なら、

> **nginx + Authelia + Remote-User**

が最もバランスのよい設計です。

---

# 実際に組むならこの構成

```text
                     Internet
                         │
                    TCP 80/443
                         │
                 ┌───────▼───────┐
                 │     nginx     │
                 │               │
                 │ TLS           │
                 │ auth_request  │
                 │ rate limit    │
                 │ access log    │
                 └───┬───────┬───┘
                     │       │
              /auth  │       │ /app
                     ▼       │
                ┌─────────┐  │
                │Authelia │  │
                └────┬────┘  │
                     │       │
                ┌────▼────┐  │
                │ users   │  │
                │ SQLite  │  │
                └─────────┘  │
                             │
            Remote-User: alice
                             │
                             ▼
                       ┌──────────┐
                       │   App    │
                       │ :8080    │
                       └──────────┘

nginx : public
Authelia : internal + nginx経由のみ
App : internal only
SQLite/users.yml : filesystem only
```

## アプリ側で最低限実装するべきこと

認証そのものを実装する必要はありませんが、以下だけは入れた方がよいです。

1. `Remote-User` が無ければアクセス拒否
2. `Remote-User` をユーザーIDとして使用
3. nginx以外からの接続を受けない
4. `Remote-User` でユーザーを切り替え、**全てのデータアクセスをそのユーザーに紐付ける**
5. username変更を許すなら、ユーザー名ではなく不変IDを使う設計も検討

特に **4 が重要**です。Authelia/nginx が解決するのは「この人が alice である」という**認証**までです。

```text
alice は alice のデータだけ読める
bob は bob のデータだけ読める
admin は管理機能を使える
```

という**認可**は、マルチユーザーアプリ自身が担保する必要があります。

---

### 結論

公開を前提にした最小構成としては、

```text
Let's Encrypt
    ↓
nginx
    ↓ auth_request
Authelia
    ↓
Remote-User
    ↓
既存アプリ
```

を推します。

**アプリの変更点は「信頼された `Remote-User` を現在ユーザーとして扱う」のみにでき、パスワード保存・ログイン画面・セッション・MFA・brute-force対策をアプリから完全に分離できます。**

必要なら次に、**Docker Compose前提で `nginx + Authelia + 既存app` のほぼそのまま起動できる構成例（`compose.yml`、nginx.conf、Authelia configuration.yml、users.ymlまで）**に落とせます。

[1]: https://www.authelia.com/integration/proxies/nginx/ "NGINX | Integration | Authelia"
[2]: https://www.authelia.com/configuration/first-factor/file/ "File | Configuration | Authelia"
[3]: https://nginx.org/en/docs/http/ngx_http_auth_request_module.html "Module ngx_http_auth_request_module"
[4]: https://www.authelia.com/configuration/security/access-control/ "Access Control | Configuration | Authelia"
[5]: https://www.authelia.com/configuration/security/regulation/ "Regulation | Configuration | Authelia"
[6]: https://www.authelia.com/blog/authelia--traefik-setup-guide/ "Authelia + Traefik Setup Guide | Blog | Authelia"
[7]: https://www.authelia.com/configuration/session/introduction/ "Session | Configuration | Authelia"
[8]: https://www.authelia.com/integration/prologue/get-started/ "Get started | Integration | Authelia"
[9]: https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html "Module ngx_http_auth_basic_module"
[10]: https://docs.goauthentik.io/add-secure-apps/providers/proxy/forward_auth "Forward auth | authentik"
[11]: https://docs.goauthentik.io/add-secure-apps/providers/proxy/ "Proxy provider | authentik"
[12]: https://docs.goauthentik.io/users-sources/user/user_ref "User properties and attributes | authentik"
