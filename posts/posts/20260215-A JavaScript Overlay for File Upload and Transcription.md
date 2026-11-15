---
title: ファイルアップロードと文字起こしを行うためのJavaScriptオーバーレイ
description: 現在のセッショントークンを使用して、選択したファイルを文字起こしエンドポイントにブラウザオーバーレイ経由でアップロードし、オーディオの長さを検出し、テキストまたはAPIエラーを表示します。
pubDatetime: 2026-02-15T14:48:58.003Z
updatedDate: 2026-02-15T15:03:06.357Z
---

このスクリプトは、文字起こしエンドポイントにファイルをアップロードするためのブラウザベースのオーバーレイを作成する、即時実行される非同期関数です。ストリクトモードで動作し、ページから抽出したベアラートークンを使用して、バックエンドAPIと通信します。

## 主要な機能

このスクリプトは、動的にモーダルインターフェイスを構築し、安全なファイル送信を処理します。

* `#client-bootstrap`要素内のJSONペイロードから`accessToken`を取得します。
* ファイル入力、ステータス表示、結果パネルを備えたフルスクリーンオーバーレイを生成します。
* オプションで、HTMLAudioElementを使用してオーディオの長さを（ミリ秒単位で）計算します。
* 選択したファイルを`FormData`を使用して`fetch`経由で文字起こしエンドポイントに送信します。
* 返された`json.text`フィールドと生のJSONレスポンスを一緒に表示します。

エラー処理は、トークンの解析、アップロードの失敗、JSONの抽出など、重要な各ステップで実装されています。インターフェイスは、Escキー、オーバーレイのクリック、または閉じるボタンを使用して閉じることができます。全体として、このスクリプトは、スタンドアロンのクライアント側のアップロードおよび文字起こしユーティリティを提供します。

```javascript
(async () => { 
  "use strict"; 
 
  // ---- 設定 ---- 
  const UPLOAD_PATH = location.origin + "/backend-api/transcribe"; 
  const ACCEPT = "*/*"; // 任意のファイルタイプを受け入れる 
 
  // ---- ヘルパー関数 ---- 
  const getAccessToken = () => { 
    const bootstrapEl = document.getElementById("client-bootstrap"); 
    if (!bootstrapEl) { 
      throw new Error('要素 #client-bootstrap が見つかりません。アクセストークンを取得できません。'); 
    } 
    let data; 
    try { 
      data = JSON.parse(bootstrapEl.innerHTML); 
    } catch (e) { 
      throw new Error("#client-bootstrap からの JSON の解析に失敗しました。"); 
    } 
    const token = data?.session?.accessToken; 
    if (!token) { 
      throw new Error("accessToken が見つかりません。data.session.accessToken を確認してください。"); 
    } 
    return token; 
  }; 
 
  const formatMsFromAudioFile = async (file) => { 
    return await new Promise((resolve) => { 
      try { 
        const url = URL.createObjectURL(file); 
        const audio = document.createElement("audio"); 
        audio.preload = "metadata"; 
        audio.src = url; 
        audio.onloadedmetadata = () => { 
          URL.revokeObjectURL(url); 
          const durSec = audio.duration; 
          if (!Number.isFinite(durSec) || durSec <= 0) return resolve(null); 
          resolve(String(durSec * 1000)); 
        }; 
        audio.onerror = () => { 
          URL.revokeObjectURL(url); 
          resolve(null); 
        }; 
      } catch { 
        resolve(null); 
      } 
    }); 
  }; 
 
  const createOverlay = () => { 
    const overlay = document.createElement("div"); 
    overlay.id = "__webm_upload_overlay__"; 
    overlay.style.position = "fixed"; 
    overlay.style.inset = "0"; 
    overlay.style.zIndex = "2147483647"; 
    overlay.style.background = "#ffffff"; 
    overlay.style.display = "flex"; 
    overlay.style.alignItems = "center"; 
    overlay.style.justifyContent = "center"; 
    overlay.style.padding = "24px"; 
 
    const panel = document.createElement("div"); 
    panel.style.width = "min(720px, 92vw)"; 
    panel.style.background = "#ffffff"; 
    panel.style.color = "#111"; 
    panel.style.border = "1px solid rgba(0,0,0,0.15)"; 
    panel.style.borderRadius = "12px"; 
    panel.style.boxShadow = "0 12px 48px rgba(0,0,0,0.15)"; 
    panel.style.padding = "16px"; 
    panel.style.fontFamily = "ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial"; 
    panel.style.lineHeight = "1.4"; 
 
    const title = document.createElement("div"); 
    title.textContent = "ファイルを選択してアップロード"; 
    title.style.fontSize = "16px"; 
    title.style.fontWeight = "600"; 
    title.style.marginBottom = "10px"; 
 
    const desc = document.createElement("div"); 
    desc.textContent = "ファイルを選択 → 送信。レスポンスの JSON テキストフィールドが表示されます。"; 
    desc.style.opacity = "0.85"; 
    desc.style.marginBottom = "12px"; 
 
    const fileRow = document.createElement("div"); 
    fileRow.style.display = "flex"; 
    fileRow.style.gap = "10px"; 
    fileRow.style.alignItems = "center"; 
    fileRow.style.marginBottom = "12px"; 
 
    const fileInput = document.createElement("input"); 
    fileInput.type = "file"; 
    fileInput.accept = ACCEPT; 
    fileInput.style.flex = "1"; 
    fileInput.style.padding = "8px"; 
    fileInput.style.background = "#ffffff"; 
    fileInput.style.color = "#111"; 
    fileInput.style.border = "1px solid rgba(0,0,0,0.2)"; 
    fileInput.style.borderRadius = "8px"; 
 
    const sendBtn = document.createElement("button"); 
    sendBtn.textContent = "送信"; 
    sendBtn.disabled = true; 
    sendBtn.style.padding = "8px 12px"; 
    sendBtn.style.borderRadius = "8px"; 
    sendBtn.style.border = "1px solid rgba(0,0,0,0.2)"; 
    sendBtn.style.background = "#f0f0f0"; 
    sendBtn.style.color = "#111"; 
    sendBtn.style.cursor = "pointer"; 
 
    const cancelBtn = document.createElement("button"); 
    cancelBtn.textContent = "閉じる"; 
    cancelBtn.style.padding = "8px 12px"; 
    cancelBtn.style.borderRadius = "8px"; 
    cancelBtn.style.border = "1px solid rgba(0,0,0,0.2)"; 
    cancelBtn.style.background = "transparent"; 
    cancelBtn.style.color = "#111"; 
    cancelBtn.style.cursor = "pointer"; 
 
    const btnRow = document.createElement("div"); 
    btnRow.style.display = "flex"; 
    btnRow.style.gap = "10px"; 
    btnRow.style.justifyContent = "flex-end"; 
    btnRow.style.marginTop = "8px"; 
 
    const status = document.createElement("div"); 
    status.style.marginTop = "12px"; 
    status.style.padding = "10px"; 
    status.style.background = "#f5f5f5"; 
    status.style.border = "1px solid rgba(0,0,0,0.12)"; 
    status.style.borderRadius = "10px"; 
    status.style.whiteSpace = "pre-wrap"; 
    status.style.wordBreak = "break-word"; 
    status.style.maxHeight = "150px"; 
    status.style.overflowY = "auto"; 
    status.textContent = "ファイルが選択されていません"; 
 
    const result = document.createElement("div"); 
    result.style.marginTop = "12px"; 
    result.style.padding = "10px"; 
    result.style.background = "#f5f5f5"; 
    result.style.border = "1px solid rgba(0,0,0,0.12)"; 
    result.style.borderRadius = "10px"; 
    result.style.whiteSpace = "pre-wrap"; 
    result.style.wordBreak = "break-word"; 
    result.style.maxHeight = "300px"; 
    result.style.overflowY = "auto"; 
    result.textContent = ""; 
 
    const close = () => { 
      overlay.remove(); 
      window.removeEventListener("keydown", onKeyDown, true); 
    }; 
 
    const onKeyDown = (e) => { 
      if (e.key === "Escape") close(); 
    }; 
 
    overlay.addEventListener("click", (e) => { 
      if (e.target === overlay) close(); 
    }); 
 
    cancelBtn.addEventListener("click", close); 
    window.addEventListener("keydown", onKeyDown, true); 
 
    fileInput.addEventListener("change", () => { 
      const f = fileInput.files?.[0]; 
      if (!f) { 
        sendBtn.disabled = true; 
        status.textContent = "ファイルが選択されていません"; 
        return; 
      } 
      sendBtn.disabled = false; 
      status.textContent = `選択: ${f.name} (${Math.round(f.size / 1024)} KB)`; 
      result.textContent = ""; 
    }); 
 
    panel.appendChild(title); 
    panel.appendChild(desc); 
    fileRow.appendChild(fileInput); 
    fileRow.appendChild(sendBtn); 
    panel.appendChild(fileRow); 
    btnRow.appendChild(cancelBtn); 
    panel.appendChild(btnRow); 
    panel.appendChild(status); 
    panel.appendChild(result); 
    overlay.appendChild(panel); 
    document.documentElement.appendChild(overlay); 
 
    return { overlay, fileInput, sendBtn, status, result, close }; 
  }; 
 
  const existing = document.getElementById("__webm_upload_overlay__"); 
  if (existing) existing.remove(); 
 
  const ui = createOverlay(); 
 
  ui.sendBtn.addEventListener("click", async () => { 
    const file = ui.fileInput.files?.[0]; 
    if (!file) return; 
 
    ui.sendBtn.disabled = true; 
    ui.status.textContent = "アップロード中..."; 
 
    try { 
      const accessToken = getAccessToken(); 
 
      const fd = new FormData(); 
      fd.append("file", file, file.name); 
 
      const durationMs = await formatMsFromAudioFile(file); 
      if (durationMs != null) { 
        fd.append("duration_ms", durationMs); 
      } 
 
      const res = await fetch(UPLOAD_PATH, { 
        method: "POST", 
        headers: { 
          authorization: "Bearer " + accessToken, 
        }, 
        body: fd, 
      }); 
 
      if (!res.ok) { 
        const bodyText = await res.text().catch(() => ""); 
        throw new Error(`HTTP ${res.status} ${res.statusText}\n${bodyText}`); 
      } 
 
      const json = await res.json(); 
      const text = json?.text; 
 
      ui.status.textContent = "完了"; 
      ui.result.textContent = 
        (typeof text === "string" ? text : "(json.text が見つかりません)") + 
        "\n\n--- 生の JSON ---\n" + 
        JSON.stringify(json, null, 2); 
    } catch (e) { 
      ui.status.textContent = "エラー"; 
      ui.result.textContent = String(e?.message || e); 
    } finally { 
      const f = ui.fileInput.files?.[0]; 
      ui.sendBtn.disabled = !f; 
    } 
  }); 
})(); 
```
