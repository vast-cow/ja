---
pubDatetime: 2026-03-30T18:49:05+09:00
title: "rcloneでGoogle Driveにアクセスするためのclient_idの作成方法"
description: "rcloneを使うとGoogle Driveにアクセスできて便利ですが、デフォルトのclientidが共有されているため、パフォーマンスが極度に低くなります。独自のclientidを作成して、この制限をなくしましょう。 https://rclone.org/drive/#making-your-ow…"
---

rcloneを使うとGoogle Driveにアクセスできて便利ですが、デフォルトのclient_idが共有されているため、パフォーマンスが極度に低くなります。独自のclient_idを作成して、この制限をなくしましょう。

[https://rclone.org/drive/#making-your-own-client-id](https://rclone.org/drive/#making-your-own-client-id) の翻訳と一部URL追加など編集してあります。

## 独自の client_id の作成方法

rclone を Google Drive のデフォルト設定で使用する場合、rclone の client_id を使用しています。これはすべての rclone ユーザー間で共有されています。Google によって、各 client_id が 1 秒あたりに実行できるクエリ数にはグローバルなレート制限が設定されています。rclone にはすでに高いクォータが割り当てられており、今後も Google に連絡して十分な水準を維持します。

デフォルトの rclone の client ID は多くのユーザーに利用されているため、独自の client ID を使用することを強く推奨します。複数のサービスを実行している場合は、それぞれのサービスごとに API キーを使用することを推奨します。Google のデフォルトのクォータは 1 秒あたり 10 トランザクションなので、それを超えないようにすることが推奨されます。これを超えると、rclone によってレート制限がかかり、処理が遅くなります。

以下は、rclone 用の独自の Google Drive client ID を作成する方法です：

1. Google アカウントで [Google API Console](https://console.developers.google.com/) にログインします。使用する Google アカウントはどれでも構いません。（アクセスしたい Google Drive のアカウントと同一である必要はありません）

2. プロジェクトを選択するか、新しいプロジェクトを作成します。

3. [APIライブラリ](https://console.cloud.google.com/apis/library) で「Drive」を検索し、「Google Drive API」を有効化します。

4. 左側のパネルで [認証情報](https://console.cloud.google.com/apis/credentials) をクリックします（「Create credentials」ではありません。こちらはウィザードが開きます）。

5. すでに「OAuth同意画面」を設定している場合は次のステップへ進みます。未設定の場合は、右パネル右上付近の「CONFIGURE CONSENT SCREEN」ボタンをクリックし、「Get started」をクリックします。次の画面で「Application name」（「rclone」で可）を入力し、「User Support Email」（自分のメールで可）を入力します。続いて [対象](https://console.cloud.google.com/auth/audience) で「External」を選択し、連絡先情報を入力して規約に同意し、「Create」をクリックします。画面左上のボックスに rclone（またはプロジェクト名）が表示されるはずです。

   （追記：GSuite ユーザーの場合は、「External」の代わりに「Internal」を選択することも可能ですが、この場合 API の利用は組織内の Google Workspace ユーザーに限定されます。）

   また、以下を含むいくつかのスコープを追加する必要があります：

   * `https://www.googleapis.com/auth/docs`
   * `https://www.googleapis.com/auth/drive`（RClone でファイルの編集・作成・削除を行うため）
   * `https://www.googleapis.com/auth/drive.metadata.readonly`（必要に応じて）

   これを行うには、左側パネルの「Data Access」をクリックし、「add or remove scopes」をクリックして上記 3 つを選択し、「update」を押します。もしくは「Manually add scopes」テキストボックス（下にスクロール）に
   `https://www.googleapis.com/auth/docs,https://www.googleapis.com/auth/drive,https://www.googleapis.com/auth/drive.metadata.readonly`
   を入力し、「add to table」を押してから「update」をクリックします。

   Data access ページに 3 つのスコープが表示されたら、下部の「save」をクリックしてください。

6. スコープ追加後、[対象](https://console.cloud.google.com/auth/audience) をクリックし、スクロールして「+ Add users」をクリックします。テストユーザーとして自分を追加し、「save」を押します。

7. 左側パネルの [概要](https://console.cloud.google.com/auth/overview) に移動し、「Create OAuth client」をクリックします。アプリケーションタイプは「デスクトップアプリ」を選択し、「Create」をクリックします（名前はデフォルトのままで構いません）。

8. client ID と client secret が表示されます。これらを控えておきます。（ステップ 5 で「External」を選択した場合はステップ 9 に進みます。「Internal」を選択した場合は公開不要なのでステップ 10 に進んでください。ただし、接続先のドライブは同じ Google Workspace 内である必要があります。）

9. [対象](https://console.cloud.google.com/auth/audience) に移動し、「アプリを公開」ボタンをクリックして確認します。まだ追加していない場合は、自分をテストユーザーとして追加します。

10. 控えておいた client ID と client secret を rclone に設定します。

最近 Google によって導入された「強化されたセキュリティ」により、本来は「アプリの検証申請」を行い、数週間待つ必要がありますが、実際にはそのまま client ID と client secret を rclone で使用できます。この場合、rclone がトークン ID を取得する際にブラウザ上で警告の強い確認画面が表示されますが、これはリモート設定時のみなので大きな問題ではありません。アプリを「Testing」状態のまま使用することも可能ですが、この場合は認可が 1 週間で期限切れになるため、頻繁な更新が必要になる点が不便です。短い有効期間が問題でなければ、Testing モードのままでも十分利用できます。

（これらの手順は GitHub の @balazer によるものです。）

Google API Console で OAuth 同意画面の作成時に「The request failed because changes to one of the field of the resource is not supported」というエラーが出る場合があります。その場合の回避策として、[Python Quickstart](https://developers.google.com/drive/api/v3/quickstart/python) ページから必要な Google Drive API キーを作成できます。「Enable the Drive API」ボタンを押すだけで Client ID と Secret を取得できます。この操作により API Console に新しいプロジェクトが自動作成される点に注意してください。
