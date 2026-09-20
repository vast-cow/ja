---
pubDatetime: 2026-01-28T16:40:49+09:00
title: "Codex アシスタントの返信と PR メッセージをクリップボードにコピーするブックマークレット"
description: "概要 この内容は、Codex の Web UI 向けに設計された JavaScript の ブックマークレットについて説明しています。その目的は、Codex のタスクからテキスト—具体的には アシスタントの返信 と プルリクエスト（PR）メッセージ—を収集し、各項目をワンクリックでクリップボードにコ…"
---

## 概要

この内容は、Codex の Web UI 向けに設計された JavaScript の **ブックマークレット**について説明しています。その目的は、Codex のタスクからテキスト—具体的には **アシスタントの返信** と **プルリクエスト（PR）メッセージ**—を収集し、各項目をワンクリックでクリップボードにコピーできるシンプルなページ内オーバーレイとして表示することです。

複数ターンにまたがって手動でテキストを選択する代わりに、このブックマークレットはページ内の関連するタスクデータを自動的に見つけ、テキストコンテンツを抽出し、PR メッセージを見やすく整形したうえで、コンパクトな「コピーリスト」インターフェースを提供します。

## ブックマークレットが抽出する内容

このスクリプトは、タスクの各ターンから Markdown 風のテキストスニペットのリストを構築します。

### アシスタントおよびユーザーメッセージのテキスト

各ターンについて、ターン内のアイテムを検査します。

* アイテムが `message` の場合、コンテンツブロックを走査し、**テキストブロックのみ**（`content_type == "text"`）を連結します。
* テキスト以外のコンテンツタイプは無視されます。

これにより、余分な UI アーティファクトのない、きれいにコピー可能なテキストが得られます。

### PR タイトルおよび PR 説明

アイテムが `pr` の場合、次の形式で整形されます。

* PR タイトルを用いた Markdown 見出し：`# <pr_title>`
* 続いて PR 本文：`<pr_message>`

これにより、コピーした PR コンテンツをドキュメント、GitHub、レビュー用メモなどでそのまま利用できます。

## ページ内で正しいデータを見つける方法

ブックマークレットにおける主な課題は、実行時オブジェクトの奥深くに埋もれたアプリケーション状態を特定することです。このスクリプトは、再帰的なオブジェクト検索ユーティリティを用いてこれを実現します。

### 循環参照を考慮したディープサーチ

関数 `findFirstParentWhere(obj, matcher)` は、次の安全策を備えた深さ優先探索をオブジェクトグラフ全体に対して実行します。

* 循環参照による無限ループを避けるために `WeakSet` を使用
* オブジェクト以外および関数をスキップ
* 訪問した各オブジェクトに対して、`matcher(parent, path)` 述語でチェック

### タスクおよびターンマッピングの特定

ブックマークレットは URL パス（最後のセグメント）から現在の `taskId` を特定し、次の条件を満たすオブジェクトを検索します。

* `task.id == taskId` となる `task` プロパティを持つオブジェクト
* すべてのキーが `taskId` で始まる `turn_mapping` プロパティを持つオブジェクト

見つかったら、次のような環境メタデータを抽出します。

* リポジトリラベル（`environment_label`）
* ブランチ名（`branch_name`）

これらはデバッグおよび確認のためにログ出力されます。

## コピーリストの順序付けと構築

会話の流れを保持するため、ターンは `created_at` に基づいて時系列順にソートされます。その後、スクリプトはターンをフラットな文字列配列（`turnMdList`）に集約します。

* ロールに応じて正しいフィールドを選択：

  * `user` → `input_items`
  * `assistant` → `output_items`
* 前述の方法でメッセージテキストおよび PR コンテンツを抽出

最終的に、コピー可能なテキストブロックの順序付き配列が生成されます。

## コピー用オーバーレイ UI

ブックマークレットは、次を含むフルスクリーンのオーバーレイを注入します。

* 暗い背景（クリックするとオーバーレイが閉じる）
* 中央に配置されたパネル：

  * ヘッダー（「Copy list (N)」）と閉じるボタン
  * スクロール可能なエントリーリスト
* トースト通知（「Copied!」など）

各行には次が表示されます。

* インデックス番号
* 抽出されたテキスト（可読性のため `<pre>` ブロック内）
* そのエントリーをコピーする **Copy** ボタン

### クリップボード処理

コピーは次の 2 段階戦略を使用します。

1. `navigator.clipboard.writeText()` を優先（モダンブラウザ）
2. 非表示の `<textarea>` を使用した `document.execCommand("copy")` へのフォールバック

これにより、ブラウザや権限設定の違いによる互換性が向上します。

### キーボードおよび操作

* **Escape** キーでオーバーレイを閉じる
* パネル外をクリックすると閉じる
* トーストメッセージや一時的なボタンラベル変更（「Copied」）による視覚的フィードバック

## これが有用な理由

このブックマークレットは、Codex タスク向けの実用的な「エクスポート」ツールです。

* アシスタントの返信を素早くコピーして共有やドキュメント化が可能
* GitHub 向けに一貫した形式で PR メッセージを抽出
* 多数のターンにわたる手動選択やスクロールを回避
* 軽量な UI 拡張として完全にクライアントサイドで動作

要するに、複雑なページ状態を、開発者のワークフローに最適化されたシンプルで信頼性の高い「コピーメニュー」へと変換します。

```javascript
javascript:!function(){const taskId=location.pathname.split(%22/%22).filter(Boolean).pop();function findFirstParentWhere(obj,matcher){const visited=new WeakSet;return function dfs(node,path){if(null===node)return null;if(%22object%22!==typeof node)return null;if(%22function%22==typeof node)return null;const anyNode=node;if(visited.has(anyNode))return null;visited.add(anyNode);if(matcher(anyNode,path))return{parent:anyNode,path:path};if(Array.isArray(anyNode))for(let i=0;i<anyNode.length;i++){const hit=dfs(anyNode[i],path.concat(i));if(hit)return hit}else for(const[k,v]of Object.entries(anyNode)){const hit=dfs(v,path.concat(k));if(hit)return hit}return null}(obj,[])}const resultFindTask=findFirstParentWhere(window,(parent,path)=>{if(!Object.prototype.hasOwnProperty.call(parent,%22task%22))return!1;const task=parent.task;return task&&task.id==taskId});if(!resultFindTask)throw new Error(%22No matching element found (taskId=%22+JSON.stringify(taskId)+%22).%22);const taskInfo=resultFindTask.parent;console.log(taskInfo);const resultTurnMapping=findFirstParentWhere(window,(parent,path)=>{if(!Object.prototype.hasOwnProperty.call(parent,%22turn_mapping%22))return!1;const turnMapping=parent.turn_mapping;return Object.keys(turnMapping).reduce((previousValue,currentValue,currentIndex,array)=>previousValue&&currentValue.startsWith(taskId),!0)});if(!resultTurnMapping)throw new Error(%22No matching element found (taskId=%22+JSON.stringify(taskId)+%22).%22);const turnsInfo=resultTurnMapping.parent;console.log(turnsInfo);const repo=taskInfo.task.task_status_display.environment_label;const branchName=taskInfo.task.task_status_display.branch_name;console.log({taskId:taskId,repo:repo,branchName:branchName,taskInfo:taskInfo,turnsInfo:turnsInfo});const turnMdList=Object.entries(turnsInfo.turn_mapping).sort(([,a],[,b])=>a.turn.created_at-b.turn.created_at).reduce((previousValue0,[,currentValue0],currentIndex0,array0)=>{let ioitemsStr=null;console.log({currentValue0:currentValue0});if(%22user%22==currentValue0.turn.role)ioitemsStr=%22input_items%22;else{if(%22assistant%22!=currentValue0.turn.role)throw Error(%22turn.role = %22+JSON.stringify(currentValue0.turn.role));ioitemsStr=%22output_items%22}return currentValue0.turn[ioitemsStr].reduce((previousValue1,currentValue1,currentIndex1,array1)=>{if(%22message%22==currentValue1.type){previousValue1.push(currentValue1.content.reduce((previousValue2,currentValue2,currentIndex2,array2)=>%22text%22==currentValue2.content_type?previousValue2+currentValue2.text:previousValue2,%22%22));return%20previousValue1}if(%22pr%22==currentValue1.type){const%20prMessage=%22#%20%22+currentValue1.pr_title+%22\n\n%22+currentValue1.pr_message;previousValue1.push(prMessage);return%20previousValue1}return%20previousValue1},previousValue0)},[]);console.log(turnMdList);!function(strings){if(!Array.isArray(strings))throw%20new%20TypeError(%22strings%20must%20be%20an%20array%22);strings=strings.map(s=%3Enull==s?%22%22:String(s));const%20existing=document.getElementById(%22__copy_overlay_root__%22);existing&&existing.remove();const%20root=document.createElement(%22div%22);root.id=%22__copy_overlay_root__%22;root.style.cssText=[%22position:fixed%22,%22inset:0%22,%22z-index:2147483647%22,%22font-family:system-ui,-apple-system,Segoe%20UI,Roboto,Helvetica,Arial,'Apple%20Color%20Emoji','Segoe%20UI%20Emoji'%22].join(%22;%22);const%20backdrop=document.createElement(%22div%22);backdrop.style.cssText=[%22position:absolute%22,%22inset:0%22,%22background:rgba(0,0,0,.45)%22].join(%22;%22);const%20panel=document.createElement(%22div%22);panel.style.cssText=[%22position:absolute%22,%22top:5vh%22,%22left:50%25%22,%22transform:translateX(-50%25)%22,%22width:min(900px,92vw)%22,%22max-height:90vh%22,%22background:#fff%22,%22border-radius:14px%22,%22box-shadow:0%2010px%2030px%20rgba(0,0,0,.25)%22,%22display:flex%22,%22flex-direction:column%22,%22overflow:hidden%22].join(%22;%22);const%20header=document.createElement(%22div%22);header.style.cssText=[%22padding:12px%2014px%22,%22display:flex%22,%22align-items:center%22,%22justify-content:space-between%22,%22border-bottom:1px%20solid%20#eee%22,%22gap:10px%22].join(%22;%22);const%20title=document.createElement(%22div%22);title.textContent=%60Copy%20list%20(${strings.length})%60;title.style.cssText=%22font-weight:600;font-size:14px;color:#111;%22;const%20closeBtn=document.createElement(%22button%22);closeBtn.type=%22button%22;closeBtn.textContent=%22%C3%97%22;closeBtn.setAttribute(%22aria-label%22,%22Close%22);closeBtn.style.cssText=[%22width:32px%22,%22height:32px%22,%22border-radius:10px%22,%22border:1px%20solid%20#ddd%22,%22background:#fff%22,%22cursor:pointer%22,%22font-size:20px%22,%22line-height:1%22].join(%22;%22);const%20body=document.createElement(%22div%22);body.style.cssText=[%22padding:10px%22,%22overflow:auto%22].join(%22;%22);const%20list=document.createElement(%22div%22);list.style.cssText=%22display:flex;flex-direction:column;gap:8px;%22;const%20toast=document.createElement(%22div%22);toast.style.cssText=[%22position:absolute%22,%22bottom:12px%22,%22left:50%25%22,%22transform:translateX(-50%25)%22,%22background:rgba(17,17,17,.92)%22,%22color:#fff%22,%22padding:8px%2010px%22,%22border-radius:10px%22,%22font-size:12px%22,%22opacity:0%22,%22transition:opacity%20.15s%20ease%22,%22pointer-events:none%22].join(%22;%22);let%20toastTimer=null;function%20showToast(msg){toast.textContent=msg;toast.style.opacity=%221%22;clearTimeout(toastTimer);toastTimer=setTimeout(()=%3Etoast.style.opacity=%220%22,900)}function%20makeRow(text,idx){const%20row=document.createElement(%22div%22);row.style.cssText=[%22display:flex%22,%22gap:10px%22,%22align-items:flex-start%22,%22border:1px%20solid%20#eee%22,%22border-radius:12px%22,%22padding:10px%22,%22background:#fafafa%22].join(%22;%22);const%20num=document.createElement(%22div%22);num.textContent=String(idx+1);num.style.cssText=[%22min-width:26px%22,%22text-align:right%22,%22color:#666%22,%22font-size:12px%22,%22padding-top:2px%22].join(%22;%22);const%20pre=document.createElement(%22pre%22);pre.textContent=text;pre.style.cssText=[%22margin:0%22,%22white-space:pre-wrap%22,%22word-break:break-word%22,%22flex:1%22,%22font-size:13px%22,%22line-height:1.35%22,%22color:#111%22].join(%22;%22);const%20actions=document.createElement(%22div%22);actions.style.cssText=%22display:flex;flex-direction:column;gap:6px;%22;const%20btn=document.createElement(%22button%22);btn.type=%22button%22;btn.textContent=%22Copy%22;btn.style.cssText=[%22border:1px%20solid%20#ddd%22,%22background:#fff%22,%22border-radius:10px%22,%22padding:7px%2010px%22,%22cursor:pointer%22,%22font-size:12px%22,%22font-weight:600%22].join(%22;%22);btn.addEventListener(%22click%22,async()=%3E{btn.disabled=!0;const%20ok=await%20async%20function(text){try{if(navigator.clipboard&&navigator.clipboard.writeText){await%20navigator.clipboard.writeText(text);return!0}}catch(_){}try{const%20ta=document.createElement(%22textarea%22);ta.value=text;ta.setAttribute(%22readonly%22,%22%22);ta.style.cssText=%22position:fixed;left:-9999px;top:-9999px;%22;document.body.appendChild(ta);ta.select();const%20ok=document.execCommand(%22copy%22);ta.remove();return%20ok}catch(_){return!1}}(text);if(ok){showToast(%22Copied!%22);btn.textContent=%22Copied%22;setTimeout(()=%3Ebtn.textContent=%22Copy%22,800)}else%20showToast(%22Copy%20failed%20(browser%20blocked).%22);btn.disabled=!1});actions.appendChild(btn);row.appendChild(num);row.appendChild(pre);row.appendChild(actions);return%20row}strings.forEach((s,i)=%3Elist.appendChild(makeRow(s,i)));function%20close(){window.removeEventListener(%22keydown%22,onKeyDown,!0);root.remove()}function%20onKeyDown(e){%22Escape%22===e.key&&close()}backdrop.addEventListener(%22click%22,close);closeBtn.addEventListener(%22click%22,close);window.addEventListener(%22keydown%22,onKeyDown,!0);header.appendChild(title);header.appendChild(closeBtn);body.appendChild(list);panel.appendChild(header);panel.appendChild(body);root.appendChild(backdrop);root.appendChild(panel);root.appendChild(toast);document.body.appendChild(root);showToast(%22Click%20Copy%20to%20copy%20text%22)}(turnMdList)}();
```

```javascript
(function () {

  const taskId = location.pathname.split("/").filter(Boolean).pop();

  function findFirstParentWhere(obj, matcher) {
    const visited = new WeakSet();

    function dfs(node, path) {
      if (node === null) return null;

      const t = typeof node;
      if (t !== "object") return null;

      if (typeof node === "function") return null;

      const anyNode = node;

      if (visited.has(anyNode)) return null;
      visited.add(anyNode);

      if (matcher(anyNode, path)) {
        return { parent: anyNode, path };
      }

      if (Array.isArray(anyNode)) {
        for (let i = 0; i < anyNode.length; i++) {
          const hit = dfs(anyNode[i], path.concat(i));
          if (hit) return hit;
        }
      } else {
        for (const [k, v] of Object.entries(anyNode)) {
          const hit = dfs(v, path.concat(k));
          if (hit) return hit;
        }
      }

      return null;
    }

    return dfs(obj, []);
  }

  function showCopyOverlay(strings) {
    if (!Array.isArray(strings)) throw new TypeError("strings must be an array");
    strings = strings.map(s => (s == null ? "" : String(s)));

    const existing = document.getElementById("__copy_overlay_root__");
    if (existing) existing.remove();

    const root = document.createElement("div");
    root.id = "__copy_overlay_root__";
    root.style.cssText = [
      "position:fixed",
      "inset:0",
      "z-index:2147483647",
      "font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial,'Apple Color Emoji','Segoe UI Emoji'",
    ].join(";");

    const backdrop = document.createElement("div");
    backdrop.style.cssText = [
      "position:absolute",
      "inset:0",
      "background:rgba(0,0,0,.45)",
    ].join(";");

    const panel = document.createElement("div");
    panel.style.cssText = [
      "position:absolute",
      "top:5vh",
      "left:50%",
      "transform:translateX(-50%)",
      "width:min(900px,92vw)",
      "max-height:90vh",
      "background:#fff",
      "border-radius:14px",
      "box-shadow:0 10px 30px rgba(0,0,0,.25)",
      "display:flex",
      "flex-direction:column",
      "overflow:hidden",
    ].join(";");

    const header = document.createElement("div");
    header.style.cssText = [
      "padding:12px 14px",
      "display:flex",
      "align-items:center",
      "justify-content:space-between",
      "border-bottom:1px solid #eee",
      "gap:10px",
    ].join(";");

    const title = document.createElement("div");
    title.textContent = `Copy list (${strings.length})`;
    title.style.cssText = "font-weight:600;font-size:14px;color:#111;";

    const closeBtn = document.createElement("button");
    closeBtn.type = "button";
    closeBtn.textContent = "×";
    closeBtn.setAttribute("aria-label", "Close");
    closeBtn.style.cssText = [
      "width:32px",
      "height:32px",
      "border-radius:10px",
      "border:1px solid #ddd",
      "background:#fff",
      "cursor:pointer",
      "font-size:20px",
      "line-height:1",
    ].join(";");

    const body = document.createElement("div");
    body.style.cssText = [
      "padding:10px",
      "overflow:auto",
    ].join(";");

    const list = document.createElement("div");
    list.style.cssText = "display:flex;flex-direction:column;gap:8px;";

    const toast = document.createElement("div");
    toast.style.cssText = [
      "position:absolute",
      "bottom:12px",
      "left:50%",
      "transform:translateX(-50%)",
      "background:rgba(17,17,17,.92)",
      "color:#fff",
      "padding:8px 10px",
      "border-radius:10px",
      "font-size:12px",
      "opacity:0",
      "transition:opacity .15s ease",
      "pointer-events:none",
    ].join(";");

    let toastTimer = null;
    function showToast(msg) {
      toast.textContent = msg;
      toast.style.opacity = "1";
      clearTimeout(toastTimer);
      toastTimer = setTimeout(() => (toast.style.opacity = "0"), 900);
    }

    async function copyText(text) {
      try {
        if (navigator.clipboard && navigator.clipboard.writeText) {
          await navigator.clipboard.writeText(text);
          return true;
        }
      } catch (_) {}

      // fallback: execCommand
      try {
        const ta = document.createElement("textarea");
        ta.value = text;
        ta.setAttribute("readonly", "");
        ta.style.cssText = "position:fixed;left:-9999px;top:-9999px;";
        document.body.appendChild(ta);
        ta.select();
        const ok = document.execCommand("copy");
        ta.remove();
        return ok;
      } catch (_) {
        return false;
      }
    }

    function makeRow(text, idx) {
      const row = document.createElement("div");
      row.style.cssText = [
        "display:flex",
        "gap:10px",
        "align-items:flex-start",
        "border:1px solid #eee",
        "border-radius:12px",
        "padding:10px",
        "background:#fafafa",
      ].join(";");

      const num = document.createElement("div");
      num.textContent = String(idx + 1);
      num.style.cssText = [
        "min-width:26px",
        "text-align:right",
        "color:#666",
        "font-size:12px",
        "padding-top:2px",
      ].join(";");

      const pre = document.createElement("pre");
      pre.textContent = text;
      pre.style.cssText = [
        "margin:0",
        "white-space:pre-wrap",
        "word-break:break-word",
        "flex:1",
        "font-size:13px",
        "line-height:1.35",
        "color:#111",
      ].join(";");

      const actions = document.createElement("div");
      actions.style.cssText = "display:flex;flex-direction:column;gap:6px;";

      const btn = document.createElement("button");
      btn.type = "button";
      btn.textContent = "Copy";
      btn.style.cssText = [
        "border:1px solid #ddd",
        "background:#fff",
        "border-radius:10px",
        "padding:7px 10px",
        "cursor:pointer",
        "font-size:12px",
        "font-weight:600",
      ].join(";");

      btn.addEventListener("click", async () => {
        btn.disabled = true;
        const ok = await copyText(text);
        if (ok) {
          showToast("Copied!");
          btn.textContent = "Copied";
          setTimeout(() => (btn.textContent = "Copy"), 800);
        } else {
          showToast("Copy failed (browser blocked).");
        }
        btn.disabled = false;
      });

      actions.appendChild(btn);
      row.appendChild(num);
      row.appendChild(pre);
      row.appendChild(actions);
      return row;
    }

    strings.forEach((s, i) => list.appendChild(makeRow(s, i)));

    function close() {
      window.removeEventListener("keydown", onKeyDown, true);
      root.remove();
    }

    function onKeyDown(e) {
      if (e.key === "Escape") close();
    }

    backdrop.addEventListener("click", close);
    closeBtn.addEventListener("click", close);
    window.addEventListener("keydown", onKeyDown, true);

    header.appendChild(title);
    header.appendChild(closeBtn);
    body.appendChild(list);
    panel.appendChild(header);
    panel.appendChild(body);

    root.appendChild(backdrop);
    root.appendChild(panel);
    root.appendChild(toast);
    document.body.appendChild(root);

    showToast("Click Copy to copy text");
    return { close };
  }

  const resultFindTask = findFirstParentWhere(window, (parent, path) => {
    if (!Object.prototype.hasOwnProperty.call(parent, "task")) return false;
    const task = parent.task;
    return task && task.id == taskId; // keep == for string/number compatibility
  });

  if (!resultFindTask) {
    throw new Error("No matching element found (taskId=" + JSON.stringify(taskId) + ").");
  }

  const taskInfo = resultFindTask.parent;

  console.log(taskInfo);

  const resultTurnMapping = findFirstParentWhere(window, (parent, path) => {
    if (!Object.prototype.hasOwnProperty.call(parent, "turn_mapping")) return false;
    const turnMapping = parent.turn_mapping;
    return Object.keys(turnMapping).reduce((previousValue, currentValue, currentIndex, array) => {
      return previousValue && currentValue.startsWith(taskId);
    }, true);
  });

  if (!resultTurnMapping) {
    throw new Error("No matching element found (taskId=" + JSON.stringify(taskId) + ").");
  }

  const turnsInfo = resultTurnMapping.parent;

  console.log(turnsInfo);

  const repo = taskInfo.task.task_status_display.environment_label;
  const branchName = taskInfo.task.task_status_display.branch_name;

  console.log({taskId, repo, branchName, taskInfo, turnsInfo});

  const turnsSorted = Object.entries(turnsInfo.turn_mapping).sort(([, a], [, b]) => a.turn.created_at - b.turn.created_at);

  const turnMdList = turnsSorted.reduce((previousValue0 /* [string, any] */, [, currentValue0] /* [string, any] */, currentIndex0 /* number */, array0 /* [string, any][] */) => {
  // .map(([, value] /* [string, any] */, index, array /* [string, any][] */) => {
    let ioitemsStr = null;
    console.log({currentValue0});
    if (currentValue0.turn.role == "user") {
      ioitemsStr = "input_items";
    } else if (currentValue0.turn.role == "assistant") {
      ioitemsStr = "output_items";
    } else {
      throw Error("turn.role = " + JSON.stringify(currentValue0.turn.role));
    }
    return currentValue0.turn[ioitemsStr].reduce((previousValue1, currentValue1, currentIndex1, array1) => {
      if (currentValue1.type == "message") {
        previousValue1.push(currentValue1.content.reduce((previousValue2, currentValue2, currentIndex2, array2) => {
          if (currentValue2.content_type == "text") {
            return previousValue2 + currentValue2.text;
          } else {
            return previousValue2;
          }
        }, ""));
        return previousValue1;
      } else if (currentValue1.type == "pr") {
        const prMessage = "# " + currentValue1.pr_title + "\n\n" + currentValue1.pr_message;
        previousValue1.push(prMessage);
        return previousValue1;
      } else {
        return previousValue1;
      }
    }, previousValue0);
  }, []);

  console.log(turnMdList);

  showCopyOverlay(turnMdList);

})();
```
