---
title: "Certbotで無料SSL証明書を取得する方法と、DNS-01に対応した無料DDNSサービスの比較"
description: "TXTレコードやサブサブドメインの制約を考慮し、No-IP、DuckDNS、Dynu、dynv6などの無料DDNSサービスを比較。自宅サーバーでのCertbot DNS-01認証に適したサービスを選定する。"
pubDatetime: 2026-10-20T12:55:00+09:00
---

自宅サーバーやVPSでHTTPSを利用する場合、Let's EncryptとCertbotを使えば、無料でSSL/TLS証明書を取得・更新できます。

しかし、自宅サーバーではTCP 80番ポートを開放できなかったり、複数のサブドメインを運用したかったりすることがあります。

特に、No-IPなどの無料DDNSサービスではDNSレコードの操作に制限があるため、Let's Encryptの認証方式によっては証明書を取得できません。

この記事では、次の内容を解説します。

- Certbotで利用できる認証方式（HTTP-01、DNS-01、TLS-ALPN-01）
- No-IP、DuckDNS、Dynu、dynv6などの無料DDNSサービスの比較
- TXTレコードとDNS-01認証の関係
- サブサブドメインとワイルドカードDNSの違い
- ワイルドカード証明書の取得方法
- 自宅サーバーに適したDDNSサービスの選び方

※サービスの仕様は2026年10月時点の公開情報に基づきます。

---

## 1. CertbotとLet's Encryptの仕組み

Let's Encryptは、無料のSSL/TLS証明書を発行する認証局（CA）です。

Certbotは、Let's EncryptなどのACME対応認証局と通信し、証明書の取得や更新を自動化するACMEクライアントです。

証明書を取得する際には、申請者が対象ドメインを管理していることを証明する必要があります。

この確認処理を**ACME Challenge（ドメイン認証）**と呼びます。

### ACME認証方式の比較

| 認証方式 | 使用ポート | 認証方法 | ワイルドカード証明書 |
|---|---|---|---|
| HTTP-01 | TCP 80 | HTTPで認証ファイルを公開 | × |
| DNS-01 | DNS | TXTレコードを登録 | ○ |
| TLS-ALPN-01 | TCP 443 | TLS接続で認証情報を提示 | × |

標準のCertbotで通常利用するのは、HTTP-01とDNS-01です。

TLS-ALPN-01はACMEで定義されていますが、Certbotでは標準サポートされていません。

### HTTP-01認証

HTTP-01では、認証局が以下のURLにアクセスしてドメインの管理権限を確認します。

```text
http://example.com/.well-known/acme-challenge/<TOKEN>
```

例えば、Certbotのstandaloneモードで取得する場合は次のように実行します。

```bash
sudo certbot certonly --standalone \
  -d example.com
```

このコマンドではCertbotが一時的なHTTPサーバーを起動します。

既存のWebサーバーの公開ディレクトリを使用する場合は、webrootモードも利用できます。

```bash
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com
```

HTTP-01の特徴は次のとおりです。

- DNSのTXTレコードを編集する必要がない
- 通常の無料DDNSでも利用しやすい
- 外部からTCP 80番ポートへアクセスできる必要がある
- ワイルドカード証明書には対応していない

NginxやApacheを利用していれば、それぞれのCertbotプラグインを使用する方法もあります。

### DNS-01認証

DNS-01では、対象ドメインのDNSにTXTレコードを登録してドメインの管理権限を証明します。

例えば、`example.com`の証明書を取得する場合は、次のようなレコードを作成します。

```text
名前:
_acme-challenge.example.com

種類:
TXT

値:
<ACMEから指定されたトークン>
```

手動で認証する場合のコマンドは以下です。

```bash
sudo certbot certonly --manual \
  --preferred-challenges dns \
  -d example.com
```

DNS-01には次のメリットがあります。

- TCP 80番ポートを開放する必要がない
- TCP 443番ポートも認証時には不要
- Webサーバーが外部公開されていなくても取得できる
- ワイルドカード証明書を取得できる
- DNS APIを利用すれば自動更新できる

ただし、**DNS-01を利用するには、必要なTXTレコードを登録できるDNSサービスが必要**です。

また、手動でTXTレコードを登録するだけでは無人での自動更新はできません。自動更新にはDNS APIや認証用スクリプトなどが必要です。

### TLS-ALPN-01認証

TLS-ALPN-01では、TCP 443番ポートを使用し、TLSのALPN拡張と専用の一時証明書でドメインを認証します。

HTTP-01とは異なり、TCP 80番ポートを使用しません。

ただし、Certbotはこの方式を標準サポートしていないため、必要な場合はacme.shやlegoなど別のACMEクライアントを検討します。

---

## 2. Certbotの実行方法

Certbotでは認証方式とは別に、使用するプラグインや認証方法を選択できます。

| オプション | 認証方式 | 用途 |
|---|---|---|
| `--nginx` | HTTP-01 | Nginxと連携 |
| `--apache` | HTTP-01 | Apacheと連携 |
| `--webroot` | HTTP-01 | 既存Webサーバーを利用 |
| `--standalone` | HTTP-01 | 一時HTTPサーバーを起動 |
| `--manual --preferred-challenges http` | HTTP-01 | HTTP認証を手動実行 |
| `--manual --preferred-challenges dns` | DNS-01 | TXTレコードを手動登録 |
| `--dns-cloudflare` | DNS-01 | Cloudflare APIを利用 |
| `--dns-route53` | DNS-01 | AWS Route 53を利用 |
| `--dns-google` | DNS-01 | Google Cloud DNSを利用 |
| `--dns-rfc2136` | DNS-01 | RFC2136によるDNS更新 |

DNS用プラグインは、Certbot本体とは別途インストールが必要な場合があります。

## 3. 無料DDNSサービスの比較

自宅サーバーでは、グローバルIPアドレスが変動することがあります。

そのため、IPアドレスが変わっても同じホスト名でアクセスできるDDNS（Dynamic DNS）が便利です。

ここでは以下のサービスを比較します。

- No-IP
- DuckDNS
- Dynu
- dynv6
- FreeDNS
- deSEC

比較する項目は、無料プランでのTXTレコード、サブサブドメイン、ワイルドカードDNS、CertbotによるDNS-01認証への対応です。

### 無料DDNS比較表

| サービス | TXTレコード | サブサブドメイン | ワイルドカードDNS | Certbot DNS-01 |
|---|---|---|---|---|
| No-IP | × | × | × | × |
| DuckDNS | ○ 制限あり | ○ 同一IP向け | △ 独自仕様 | ○ 外部プラグイン |
| Dynu | ○ | ○ | ○ | ○ |
| dynv6 | ○ | ○ | ○ | ○ |
| FreeDNS | ○ | △ 利用ドメイン次第 | × 無料版 | ○ 手動認証など |
| deSEC | ○ | ○ 既存ゾーン内 | ○ | ○ ※新規DDNS登録停止中 |

※○は利用可能、△は条件付き、×は非対応です。

※表は各サービスが提供する無料DDNSホスト名を対象としています。独自ドメインを持ち込む場合は条件が変わります。

※FreeDNSの無料プランでは、公式料金表でワイルドカードDNSが非対応となっています。

### No-IP

公式サイト：https://www.noip.com/

No-IPは、無料DDNSとして広く知られているサービスです。

例えば以下のようなホスト名を利用できます。

```text
myserver.ddns.net
```

ただし、無料プランには制限があります。

- TXTレコードを利用できない
- 第4階層のサブドメインを作成できない
- 無料ホスト名でワイルドカードDNSを利用できない
- 通常のDNS-01認証には利用できない

例えば次のような構成には対応していません。

```text
myserver.ddns.net
app.myserver.ddns.net
nas.myserver.ddns.net
```

一方、HTTP-01認証は利用可能です。

```bash
sudo certbot certonly --standalone \
  -d myserver.ddns.net
```

TCP 80番ポートを開放できる環境で、単一ホスト名を利用するなら選択肢になります。

No-IPの有料Enhancedプランでも第4階層サブドメインには制限があり、独自ドメインと対応DNSプランが必要になるケースがあります。

参考：[No-IP — Free and Enhanced Hostname Limitations](https://www.noip.com/support/knowledgebase/free-enhanced-limitations)

### DuckDNS

公式サイト：https://www.duckdns.org/

DuckDNSでは、以下のような無料ホスト名を取得できます。

```text
myserver.duckdns.org
```

DuckDNSはDNS-01に必要なTXTレコード更新APIを提供しています。

通常のDNSサービスとは異なり、同じDuckDNSホスト名配下のサブサブドメインはTXTレコードを共有する特殊な仕組みです。

例えば、

```text
myserver.duckdns.org
app.myserver.duckdns.org
nas.myserver.duckdns.org
```

これらの名前に対するTXT問い合わせは、DuckDNSの共有TXTレコードの仕組みを利用します。

この仕様を利用してDNS-01認証を実行できます。

ただし、TXTレコードをホスト名ごとに独立して管理できないため、複数の認証を同時に実行する際には制約があります。

参考：[DuckDNS API仕様](https://www.duckdns.org/spec.jsp)

### Dynu

公式サイト：https://www.dynu.com/

Dynuは、無料で利用できるDNS機能が充実しているDDNSサービスです。

主な特徴は次のとおりです。

- TXTレコードを利用可能
- サブサブドメインを作成可能
- ワイルドカードDNSを利用可能
- DNS APIに対応
- RFC2136によるDNS更新に対応

例えば次の構成が可能です。

```text
myserver.dynu.com
app.myserver.dynu.com
nas.myserver.dynu.com
git.myserver.dynu.com
```

ワイルドカードDNSを利用すれば、これらを同じIPアドレスへまとめて解決できます。

```text
*.myserver.dynu.com → 203.0.113.10
```

RFC2136を使用すれば、CertbotからTXTレコードを自動操作する構成も可能です。

参考：[Dynu DNS機能](https://www.dynu.com/DynamicDNS/Features)

### dynv6

公式サイト：https://dynv6.com/

dynv6はIPv4とIPv6に対応した無料DDNSサービスです。

次のようなホスト名を利用できます。

```text
myserver.dynv6.net
```

主な特徴は次のとおりです。

- A、AAAA、TXTなどのDNSレコードに対応
- ワイルドカードDNSに対応
- サブサブドメインを設定可能
- REST APIを利用可能
- RFC2136によるDNS更新に対応
- IPv6環境での運用にも適している

例えば以下のような構成を作成できます。

```text
myserver.dynv6.net
├── app.myserver.dynv6.net
├── nas.myserver.dynv6.net
└── git.myserver.dynv6.net
```

DNSレコードの操作を自動化し、CertbotのDNS-01認証と組み合わせることができます。

参考：[dynv6 APIドキュメント](https://dynv6.com/docs/apis)

### FreeDNS

公式サイト：https://freedns.afraid.org/

FreeDNSは、無料のDNSホスティングとDDNSを提供しています。

TXT、A、AAAA、CNAMEなど多くのレコード形式に対応しています。

ただし、共有ドメインを利用する場合は、利用可能な名前やレコードの編集権限に制約があります。

また、**無料プランではワイルドカードDNSを利用できません**。

DNS-01は必要なTXTレコードを設定できる場合に使用できますが、無人で自動更新するには別途更新手段の検討が必要です。

参考：[FreeDNS Membership Levels](https://freedns.afraid.org/pricing/)

### deSEC

公式サイト：https://desec.io/

deSECは、DNSSECやAPIによるDNS操作をサポートする無料DNSホスティングサービスです。

機能面ではDNS-01との相性がよく、ワイルドカード証明書の自動更新にも利用できます。

ただし、2026年10月時点で、無料DDNS用ドメイン`dedyn.io`の新規登録は停止されています。

そのため、新規に無料のDDNSホスト名を取得したい場合には、現時点では候補にしにくい状況です。

独自ドメインのDNS管理サービスとしては引き続き選択肢になります。

---

## 4. サブサブドメインとは

例えば、次のドメインがあるとします。

```text
example.com
```

その下に`app`を付けた名前がサブドメインです。

```text
app.example.com
```

さらに`test`を付けると、サブサブドメインになります。

```text
test.app.example.com
```

DDNSサービスの場合も、考え方は同じです。

```text
myserver.dynu.com
app.myserver.dynu.com
nas.myserver.dynu.com
```

`myserver.dynu.com`というDDNSホスト名から見て、`app`や`nas`はその下のサブドメインです。

DNSの階層としては、`app.myserver.dynu.com`は第4階層の名前になります。

### 同じIPアドレスでも問題ない

複数のサブサブドメインを利用する際、それぞれに異なるIPアドレスを割り当てる必要はありません。

例えば、以下のようにすべて同じIPアドレスでも構いません。

```text
app.myserver.dynu.com → 203.0.113.10
nas.myserver.dynu.com → 203.0.113.10
git.myserver.dynu.com → 203.0.113.10
```

この場合、Nginxなどのリバースプロキシでホスト名ごとに転送先を振り分けられます。

例えば、

```text
app.myserver.dynu.com → localhost:3000
nas.myserver.dynu.com → localhost:5000
git.myserver.dynu.com → localhost:8080
```

という構成です。

DNSが行うのはホスト名からIPアドレスへの解決までであり、その先のサービスの振り分けはWebサーバーやリバースプロキシが担当します。

---

## 5. ワイルドカードDNSとは

ワイルドカードDNSは、複数のサブドメインをまとめて解決する仕組みです。

通常のDNS設定では、ホスト名ごとにレコードを作成します。

```text
app.myserver.dynu.com A 203.0.113.10
nas.myserver.dynu.com A 203.0.113.10
git.myserver.dynu.com A 203.0.113.10
```

一方、ワイルドカードDNSを利用すれば、次のように設定できます。

```text
*.myserver.dynu.com A 203.0.113.10
```

これにより、個別レコードが存在しない次のような名前も同じIPアドレスに解決できます。

```text
app.myserver.dynu.com
nas.myserver.dynu.com
git.myserver.dynu.com
newapp.myserver.dynu.com
```

つまり、新しいサブドメインを追加するたびにDNSレコードを作成する必要がなくなります。

### 個別レコードとの違い

| 項目 | 個別サブサブドメイン | ワイルドカードDNS |
|---|---|---|
| 設定例 | `app.myserver.dynu.com` | `*.myserver.dynu.com` |
| ホスト名の追加 | 個別に設定 | 原則不要 |
| 同じIPにまとめる | ○ | ○ |
| 異なるIPを指定 | ○ | 個別レコードで対応 |
| 多数のサービスの運用 | 設定数が増える | 管理しやすい |

なお、DNSのワイルドカードは、既存の明示的なDNSレコードより優先されるわけではありません。また、ワイルドカードが一致するかどうかはDNS階層と既存レコードの状態によって決まります。

---

## 6. ワイルドカードDNSとワイルドカード証明書の違い

ここは特に混同しやすい部分です。

**ワイルドカードDNSとワイルドカード証明書は別の機能**です。

ワイルドカードDNSは、ホスト名をIPアドレスに解決する仕組みです。

ワイルドカード証明書は、複数のホスト名でHTTPSを利用するためのTLS証明書です。

| 項目 | ワイルドカードDNS | ワイルドカード証明書 |
|---|---|---|
| 目的 | DNS名前解決 | HTTPSの認証・暗号化 |
| 例 | `*.example.com A 203.0.113.10` | `*.example.com` |
| 新しいホスト名に対応 | DNSレコード追加が原則不要 | 証明書再発行が原則不要 |
| Certbotが必要 | × | 取得時に使用可能 |
| DNS-01が必要 | × | Let's Encryptでは必須 |

### ワイルドカード証明書の対象範囲

例えば、次の証明書を取得した場合、

```text
*.myserver.dynu.com
```

利用できるホスト名は次のとおりです。

| ホスト名 | 証明書の対象 |
|---|---|
| `app.myserver.dynu.com` | ○ |
| `nas.myserver.dynu.com` | ○ |
| `git.myserver.dynu.com` | ○ |
| `myserver.dynu.com` | × |
| `test.app.myserver.dynu.com` | × |

ワイルドカード証明書は、原則として`*`の位置にある1階層分の名前に対応します。

そのため、元のDDNSホスト名もHTTPSで利用したい場合は、証明書に両方を指定します。

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d myserver.dynu.com \
  -d '*.myserver.dynu.com'
```

このコマンドではTXTレコードを手動登録する必要があります。

また、2つの名前についてDNS-01認証が要求されるため、同じ`_acme-challenge`名に複数のTXT値が必要になる場合があります。DNSサービスが複数TXT値を扱えるか確認してください。

---

## 7. DuckDNSでCertbotのDNS-01認証を使用する

DuckDNSではTXTレコード更新用APIを利用できるため、TCP 80番ポートを開放せずに証明書を取得できます。

Certbotにはコミュニティ製のDuckDNSプラグインがあります。

### プラグインをインストール

Snap版Certbotを利用している場合の例です。

```bash
sudo snap install certbot-dns-duckdns

sudo snap connect certbot:plugin certbot-dns-duckdns
```

プラグインが認識されているか確認します。

```bash
sudo certbot plugins
```

### APIトークンを設定

DuckDNSの管理画面からAPIトークンを確認します。

認証情報ファイルを作成します。

```text
/etc/letsencrypt/duckdns.ini
```

内容は以下です。

```ini
dns_duckdns_token = YOUR_DUCKDNS_TOKEN
```

アクセス権限を制限します。

```bash
sudo chmod 600 /etc/letsencrypt/duckdns.ini
```

### 証明書を取得

```bash
sudo certbot certonly \
  --authenticator dns-duckdns \
  --dns-duckdns-credentials /etc/letsencrypt/duckdns.ini \
  --dns-duckdns-propagation-seconds 60 \
  -d myserver.duckdns.org
```

ワイルドカード証明書を取得したい場合は、対象名を次のように指定します。

```bash
sudo certbot certonly \
  --authenticator dns-duckdns \
  --dns-duckdns-credentials /etc/letsencrypt/duckdns.ini \
  --dns-duckdns-propagation-seconds 60 \
  -d '*.myserver.duckdns.org'
```

DuckDNSはTXTレコードを共有する仕様のため、複数のDNS-01チャレンジを同時に要求する構成には注意が必要です。

取得後は、更新テストを行います。

```bash
sudo certbot renew --dry-run
```

なお、Certbotで証明書を取得しても、HTTPS公開時には対象サービスに証明書を設定する必要があります。

参考：[certbot-dns-duckdns（GitHub）](https://github.com/infinityofspace/certbot_dns_duckdns)

---

## 8. Dynuやdynv6でCertbotを自動化する

Dynuとdynv6ではRFC2136によるDNS更新が利用できます。

RFC2136はDNSレコードを動的に変更するための仕組みです。

Certbotの`dns-rfc2136`プラグインと組み合わせることで、DNS-01に必要なTXTレコードを自動作成・削除できます。

基本的な流れは以下です。

1. DDNSサービスでホスト名を取得する
2. DNS更新用のTSIGキーを設定する
3. CertbotのRFC2136プラグインをインストールする
4. DNSサーバーと認証情報をCertbotに設定する
5. DNS-01認証で証明書を取得する
6. `certbot renew --dry-run`で更新をテストする

実際のDNSサーバー名、TSIGアルゴリズム、キーの形式はサービスごとに異なります。

Dynuの公式ドキュメントには、`nsupdate`を使って`_acme-challenge`のTXTレコードを作成する例が掲載されています。

dynv6もRFC2136とREST APIに対応しています。

参考：

- [Dynu NSUpdate](https://www.dynu.com/DynamicDNS/IPUpdateClient/NSUpdate)
- [dynv6 API Documentation](https://dynv6.com/docs/apis)
- [Certbot公式ユーザーガイド](https://eff-certbot.readthedocs.io/en/stable/using.html)

---

## 9. 自宅サーバーではどの構成がおすすめか

用途によって最適な構成は異なります。

| 利用条件 | おすすめ |
|---|---|
| TCP 80を開放できる | HTTP-01 |
| TCP 80を開放できない | DNS-01 |
| 単一ホスト名でよい | No-IP、DuckDNSなど |
| TXTレコードを使いたい | DuckDNS、Dynu、dynv6 |
| サブサブドメインを複数使いたい | Dynu、dynv6 |
| ワイルドカードDNSを使いたい | Dynu、dynv6 |
| ワイルドカード証明書を取得したい | DNS-01対応DDNS |
| Certbotで自動更新したい | DNS APIやRFC2136対応サービス |
| IPv6も利用したい | dynv6など |

### 推奨構成例

例えば、dynv6を利用して自宅サーバーで複数のWebサービスを公開する場合、以下の構成が考えられます。

```text
DDNS:
myserver.dynv6.net

DNS:
*.myserver.dynv6.net
        ↓
  203.0.113.10
        ↓
     Nginx
        ├── app  → localhost:3000
        ├── nas  → localhost:5000
        └── git  → localhost:8080
```

TLS証明書は、次の2つの名前を対象に取得します。

```text
myserver.dynv6.net
*.myserver.dynv6.net
```

この構成には以下のメリットがあります。

- サブサブドメインを増やしてもDNS設定を変更せずに済む
- 同じIPアドレスで複数のWebサービスを運用できる
- ワイルドカード証明書を共通利用できる
- DNS APIまたはRFC2136によって証明書更新を自動化できる

なお、DNS-01はポート開放なしで証明書を取得する方式であり、Webサービスそのものを外部公開する仕組みではありません。外部からHTTPSでアクセスするには、TCP 443への到達経路やVPN、リバーストンネルなどが別途必要です。

また、家庭用回線でCGNATが利用されている場合、DDNSだけでは外部からの直接接続を実現できない点にも注意が必要です。

---

## 10. まとめ

Certbotで無料のSSL/TLS証明書を取得する場合、通常はHTTP-01またはDNS-01を使用します。

HTTP-01は設定が比較的簡単ですが、TCP 80番ポートへの外部アクセスが必要です。

一方、DNS-01ではポート開放なしで証明書を取得でき、ワイルドカード証明書にも対応しています。

ただし、DNS-01を利用するためには、DDNSサービス側で適切なTXTレコードを登録できることが条件です。

No-IPの無料プランはHTTP-01には使えますが、TXTレコードやサブサブドメインを利用する用途には適していません。

DuckDNSは手軽にDNS-01を使える一方、TXTレコードの管理に独自の制限があります。

Dynuとdynv6は、TXTレコード、サブサブドメイン、ワイルドカードDNS、DNS更新APIに対応しているため、複数のサービスを運用する自宅サーバーに適しています。

**無料DDNSを利用し、複数のサブサブドメインを同じIPアドレスへ向け、Certbotでワイルドカード証明書を自動更新したい場合は、Dynuまたはdynv6が有力な選択肢です。**

### 参考資料

- [Certbot公式ドキュメント](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Let's Encrypt Challenge Types](https://letsencrypt.org/docs/challenge-types/)
- [No-IP — Free and Enhanced Hostname Limitations](https://www.noip.com/support/knowledgebase/free-enhanced-limitations)
- [DuckDNS API仕様](https://www.duckdns.org/spec.jsp)
- [Dynu Dynamic DNS](https://www.dynu.com/)
- [dynv6](https://dynv6.com/)
- [FreeDNS](https://freedns.afraid.org/)
- [deSEC](https://desec.io/)
- [Certbot DuckDNS Plugin](https://github.com/infinityofspace/certbot_dns_duckdns)

---

*最終確認：2026年10月20日。無料サービスの機能、利用制限、新規登録の受付状況は変更される場合があります。導入前に各サービスの公式情報を確認してください。*
