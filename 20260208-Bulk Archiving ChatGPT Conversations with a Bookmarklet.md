---
pubDatetime: 2026-02-08T21:38:43+09:00
title: "ブックマークレットでChatGPTの会話を一括アーカイブする方法"
description: "このスクリプトは、ChatGPTの履歴ページ上で複数の会話をまとめてアーカイブするための JavaScriptブックマークレット です。ブラウザ上で直接実行され、ページ遷移や拡張機能を必要としません。 全体の概要 ブックマークレットを実行すると、現在のページ上にオーバーレイUIが表示されます。 そこ…"
---

このスクリプトは、ChatGPTの履歴ページ上で複数の会話をまとめてアーカイブするための **JavaScriptブックマークレット** です。ブラウザ上で直接実行され、ページ遷移や拡張機能を必要としません。

## 全体の概要

ブックマークレットを実行すると、現在のページ上にオーバーレイUIが表示されます。
そこには会話履歴の一覧がチェックボックス付きで並び、選択した会話を一括でアーカイブできます。
処理状況はコンソールとUI内のログ領域の両方に表示されます。

## 認証情報の取得

スクリプトは、ページ内に埋め込まれている `client-bootstrap` 要素から `accessToken` を取得します。
このトークンは、ChatGPTのバックエンドAPIに対して認証付きリクエストを送るために使用されます。
取得に失敗した場合は、ログを出力して処理を中断します。

## アーカイブ処理の仕組み

`archiveConversation` 関数は、会話IDを受け取り、該当する会話をアーカイブするための `PATCH` リクエストを送信します。
レスポンスが正常で、`success: true` が返った場合のみ成功と判定されます。
各会話は順番に処理され、結果がログに記録されます。

## 会話履歴の収集

履歴サイドバー内のリンク要素をDOMから取得し、URLから会話IDを抽出します。
リンクに表示されているテキストは、そのまま会話タイトルとして使用されます。
これらの情報をもとに、UI上の一覧が生成されます。

## オーバーレイUIの構成

オーバーレイUIは画面全体を覆う半透明の背景と、中央に表示されるパネルで構成されています。
パネル内には以下の要素があります。

* UIを閉じるためのボタン
* 会話タイトルとチェックボックスの一覧
* 選択した会話をアーカイブするボタン
* 処理状況を表示するログエリア

ボタンはページ側のCSSの影響を受けないよう、ブラウザ標準に近い見た目に強制的にスタイルが指定されています。

## ログ表示とデバッグ

ログ出力用のユーティリティ関数が用意されており、
同じ内容が `console.log` とUI内のログ領域の両方に表示されます。
これにより、処理の進行状況やエラーを視覚的に確認できます。

## 終了処理

アーカイブ処理が完了すると、オーバーレイUIは自動的に削除されます。
また、途中で閉じるボタンを押した場合もUIのみが消え、ページ自体には影響を残しません。

このブックマークレットを使うことで、ChatGPTの会話管理をブラウザだけで効率的に行うことができます。

```javascript
javascript:(async()=>{let logContainer=null;function log(...args){console.log(%22[Archive Bookmarklet]%22,...args);if(logContainer){const line=document.createElement(%22div%22);line.innerText=args.map(String).join(%22 %22);logContainer.appendChild(line);logContainer.scrollTop=logContainer.scrollHeight}}log(%22start%22);const bootstrapEl=document.getElementById(%22client-bootstrap%22);if(!bootstrapEl){log(%22client-bootstrap not found%22);return}const accessToken=JSON.parse(bootstrapEl.innerHTML).session.accessToken;log(%22accessToken loaded%22);async function archiveConversation(conversationId){log(%22archive start:%22,conversationId);const response=await fetch(%22https://chatgpt.com/backend-api/conversation/%22+conversationId,{headers:{authorization:%22Bearer %22+accessToken,%22content-type%22:%22application/json%22},body:JSON.stringify({is_archived:!0}),method:%22PATCH%22,mode:%22cors%22,credentials:%22include%22});if(!response.ok){log(%22HTTP error:%22,response.status);return!1}const result=await response.json();log(%22result:%22,JSON.stringify(result));return!0===result.success}const links=Array.from(document.querySelectorAll(%22#history%20a%22));log(%22links%20found:%22,links.length);const%20conversations=links.map(a=%3E{const%20match=a.getAttribute(%22href%22)?.match(/^\/c\/(.+)$/);return%20match?{conversationId:match[1],title:a.innerText.trim()}:null}).filter(Boolean);log(%22conversations%20parsed:%22,conversations.length);function%20normalizeButtonStyle(btn){btn.style.all=%22revert%22;btn.style.font=%22inherit%22;btn.style.padding=%226px%2012px%22;btn.style.border=%221px%20solid%20#888%22;btn.style.borderRadius=%224px%22;btn.style.background=%22#eee%22;btn.style.color=%22#000%22;btn.style.cursor=%22pointer%22}const%20overlay=document.createElement(%22div%22);overlay.style.position=%22fixed%22;overlay.style.inset=%220%22;overlay.style.background=%22rgba(0,0,0,0.6)%22;overlay.style.zIndex=%2299999%22;overlay.style.display=%22flex%22;overlay.style.justifyContent=%22center%22;overlay.style.alignItems=%22center%22;const%20panel=document.createElement(%22div%22);panel.style.background=%22#fff%22;panel.style.padding=%2216px%22;panel.style.width=%22640px%22;panel.style.maxHeight=%2280%25%22;panel.style.overflow=%22auto%22;panel.style.borderRadius=%228px%22;panel.style.fontSize=%2214px%22;panel.style.position=%22relative%22;const%20closeButton=document.createElement(%22button%22);closeButton.innerText=%22%C3%97%22;normalizeButtonStyle(closeButton);closeButton.style.position=%22absolute%22;closeButton.style.top=%228px%22;closeButton.style.right=%228px%22;closeButton.onclick=()=%3E{log(%22UI%20closed%22);overlay.remove()};panel.appendChild(closeButton);const%20title=document.createElement(%22h2%22);title.innerText=%22Archive%20Conversations%22;panel.appendChild(title);const%20list=document.createElement(%22div%22);conversations.forEach(c=%3E{const%20label=document.createElement(%22label%22);label.style.display=%22block%22;label.style.marginBottom=%226px%22;const%20checkbox=document.createElement(%22input%22);checkbox.type=%22checkbox%22;checkbox.dataset.conversationId=c.conversationId;label.appendChild(checkbox);label.appendChild(document.createTextNode(%22%20%22+c.title));list.appendChild(label)});panel.appendChild(list);const%20archiveButton=document.createElement(%22button%22);archiveButton.innerText=%22Archive%20selected%22;normalizeButtonStyle(archiveButton);archiveButton.style.marginTop=%2212px%22;archiveButton.onclick=async()=%3E{if(archiveButton.disabled)return;archiveButton.disabled=!0;archiveButton.style.opacity=%220.6%22;archiveButton.style.cursor=%22not-allowed%22;archiveButton.innerText=%22Archiving...%22;const%20allCheckboxes=Array.from(panel.querySelectorAll(%22input[type=checkbox]%22));allCheckboxes.forEach(cb=%3E{cb.disabled=!0});const%20checked=allCheckboxes.filter(cb=%3Ecb.checked);log(%22selected:%22,checked.length);for(const%20cb%20of%20checked){const%20id=cb.dataset.conversationId;log(id,%22-%3E%22,await%20archiveConversation(id))}log(%22done%22);overlay.remove()};panel.appendChild(archiveButton);const%20logTitle=document.createElement(%22h3%22);logTitle.innerText=%22Log%22;logTitle.style.marginTop=%2216px%22;panel.appendChild(logTitle);logContainer=document.createElement(%22div%22);logContainer.style.background=%22#f5f5f5%22;logContainer.style.padding=%228px%22;logContainer.style.height=%22120px%22;logContainer.style.overflow=%22auto%22;logContainer.style.fontFamily=%22monospace%22;logContainer.style.fontSize=%2212px%22;panel.appendChild(logContainer);overlay.appendChild(panel);document.body.appendChild(overlay);log(%22UI%20rendered%22)})();
```

```javascript
(async () => {
  /* =============================
   * Logging utility
   * ============================= */
  let logContainer = null;

  function log(...args) {
    console.log("[Archive Bookmarklet]", ...args);

    if (logContainer) {
      const line = document.createElement("div");
      line.innerText = args.map(String).join(" ");
      logContainer.appendChild(line);
      logContainer.scrollTop = logContainer.scrollHeight;
    }
  }

  log("start");

  /* =============================
   * Load accessToken
   * ============================= */
  const bootstrapEl = document.getElementById("client-bootstrap");
  if (!bootstrapEl) {
    log("client-bootstrap not found");
    return;
  }

  const accessToken =
    JSON.parse(bootstrapEl.innerHTML).session.accessToken;

  log("accessToken loaded");

  /* =============================
   * archiveConversation
   * ============================= */
  async function archiveConversation(conversationId) {
    log("archive start:", conversationId);

    const response = await fetch(
      "https://chatgpt.com/backend-api/conversation/" + conversationId,
      {
        headers: {
          authorization: "Bearer " + accessToken,
          "content-type": "application/json"
        },
        body: JSON.stringify({ is_archived: true }),
        method: "PATCH",
        mode: "cors",
        credentials: "include"
      }
    );

    if (!response.ok) {
      log("HTTP error:", response.status);
      return false;
    }

    const result = await response.json();
    log("result:", JSON.stringify(result));

    return result.success === true;
  }

  /* =============================
   * Collect history links
   * ============================= */
  const links = Array.from(
    document.querySelectorAll("#history a")
  );

  log("links found:", links.length);

  const conversations = links
    .map((a) => {
      const match = a
        .getAttribute("href")
        ?.match(/^\/c\/(.+)$/);

      if (!match) return null;

      return {
        conversationId: match[1],
        title: a.innerText.trim()
      };
    })
    .filter(Boolean);

  log("conversations parsed:", conversations.length);

  /* =============================
   * Normalize button style
   * ============================= */
  function normalizeButtonStyle(btn) {
    btn.style.all = "revert";
    btn.style.font = "inherit";
    btn.style.padding = "6px 12px";
    btn.style.border = "1px solid #888";
    btn.style.borderRadius = "4px";
    btn.style.background = "#eee";
    btn.style.color = "#000";
    btn.style.cursor = "pointer";
  }

  /* =============================
   * Overlay UI
   * ============================= */
  const overlay = document.createElement("div");
  overlay.style.position = "fixed";
  overlay.style.inset = "0";
  overlay.style.background = "rgba(0,0,0,0.6)";
  overlay.style.zIndex = "99999";
  overlay.style.display = "flex";
  overlay.style.justifyContent = "center";
  overlay.style.alignItems = "center";

  const panel = document.createElement("div");
  panel.style.background = "#fff";
  panel.style.padding = "16px";
  panel.style.width = "640px";
  panel.style.maxHeight = "80%";
  panel.style.overflow = "auto";
  panel.style.borderRadius = "8px";
  panel.style.fontSize = "14px";
  panel.style.position = "relative";

  /* ---- Close button ---- */
  const closeButton = document.createElement("button");
  closeButton.innerText = "×";
  normalizeButtonStyle(closeButton);
  closeButton.style.position = "absolute";
  closeButton.style.top = "8px";
  closeButton.style.right = "8px";

  closeButton.onclick = () => {
    log("UI closed");
    overlay.remove();
  };

  panel.appendChild(closeButton);

  /* ---- Title ---- */
  const title = document.createElement("h2");
  title.innerText = "Archive Conversations";
  panel.appendChild(title);

  /* ---- Conversation list ---- */
  const list = document.createElement("div");

  conversations.forEach((c) => {
    const label = document.createElement("label");
    label.style.display = "block";
    label.style.marginBottom = "6px";

    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";
    checkbox.dataset.conversationId = c.conversationId;

    label.appendChild(checkbox);
    label.appendChild(
      document.createTextNode(" " + c.title)
    );

    list.appendChild(label);
  });

  panel.appendChild(list);

  /* ---- Archive button ---- */
  const archiveButton = document.createElement("button");
  archiveButton.innerText = "Archive selected";
  normalizeButtonStyle(archiveButton);
  archiveButton.style.marginTop = "12px";

  archiveButton.onclick = async () => {
    if (archiveButton.disabled) return;

    // Disable archive button
    archiveButton.disabled = true;
    archiveButton.style.opacity = "0.6";
    archiveButton.style.cursor = "not-allowed";
    archiveButton.innerText = "Archiving...";

    // Disable all checkboxes
    const allCheckboxes = Array.from(
      panel.querySelectorAll("input[type=checkbox]")
    );
    allCheckboxes.forEach((cb) => {
      cb.disabled = true;
    });

    const checked = allCheckboxes.filter((cb) => cb.checked);
    log("selected:", checked.length);

    for (const cb of checked) {
      const id = cb.dataset.conversationId;
      const ok = await archiveConversation(id);
      log(id, "->", ok);
    }

    log("done");
    overlay.remove();
  };

  panel.appendChild(archiveButton);

  /* ---- Log output ---- */
  const logTitle = document.createElement("h3");
  logTitle.innerText = "Log";
  logTitle.style.marginTop = "16px";
  panel.appendChild(logTitle);

  logContainer = document.createElement("div");
  logContainer.style.background = "#f5f5f5";
  logContainer.style.padding = "8px";
  logContainer.style.height = "120px";
  logContainer.style.overflow = "auto";
  logContainer.style.fontFamily = "monospace";
  logContainer.style.fontSize = "12px";

  panel.appendChild(logContainer);

  overlay.appendChild(panel);
  document.body.appendChild(overlay);

  log("UI rendered");
})();
```
