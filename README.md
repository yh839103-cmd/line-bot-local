# LINE bot 接本地開源模型（W6 作業）

讓 LINE bot 用**跑在自己電腦上的開源模型**（Ollama 的 Gemma 4，`gemma4:e4b`）回答問題。
需要最新資訊時，模型會自己上網搜尋，回覆最後附上來源網址。

整個 bot 只有一支 Python 程式：[`line_bot_local.py`](line_bot_local.py)（課程提供的原版）。

## 運作方式

```
手機 LINE ⇄ LINE 平台 ──Webhook──▶ Cloudflare 臨時通道 ──▶ 我電腦上的 line_bot_local.py
                                                              │
                                                              ├─▶ Ollama（gemma4:e4b，本地模型）
                                                              │     └─ 需要時呼叫 web_search／web_fetch 上網查
                                                              └─▶ 回覆 LINE（附來源網址）
```

- 程式啟動時會把 LINE 的 Webhook 換成 Cloudflare 臨時通道的網址；按 **Ctrl+C** 結束時會換回原本的網址（我的是 W5 部署在 Render 的 bot）。
- LINE 的通行碼和 Channel secret 是**執行時才輸入**，不寫進任何檔案。

## 怎麼裝（Windows / PowerShell）

1. **Python 3.10 以上**（只用內建功能，不用 `pip install`）
   ```
   python --version
   ```
2. **Ollama 和模型**（模型約 6.6GB）
   ```
   ollama pull gemma4:e4b
   ollama list
   ```
3. **cloudflared**（免費臨時通道，不用帳號）
   ```
   winget install --id Cloudflare.cloudflared -e --accept-package-agreements --accept-source-agreements
   cloudflared --version
   ```
4. **LINE 設定**
   - LINE Developers →「Basic settings」→ 複製 **Channel secret**
   - LINE Developers →「Messaging API」→ 複製 **Channel access token (long-lived)**（不要按 Reissue）
   - LINE Official Account Manager → 設定 → 回應設定：**Webhook 開啟、自動回應訊息關閉**

## 怎麼跑

```
cd "$env:USERPROFILE\Downloads\line-bot"
python line_bot_local.py
```

1. 依序貼上 LINE 長效通行碼、Channel secret（畫面不會顯示），試算表 ID 直接按 Enter 跳過
2. 等畫面出現「開好了」，用手機傳訊息給 bot
3. 結束時按 **Ctrl+C**（不要直接按視窗的 ✕），程式會把 Webhook 換回原本的網址

測試問題範例：「2026 年諾貝爾物理獎得主是誰？」→ 模型會先上網搜尋，回覆附來源。

## 我改了什麼

- **程式**：使用課程提供的原版 `line_bot_local.py`，目前沒有修改。
- **設定**：沒有安裝 gws（Google Workspace CLI），所以**照片存雲端硬碟、排日曆、寫試算表這三個功能沒有啟用**，只用「問答＋上網搜尋」。程式會顯示「找不到 gws.exe」的警告，但問答功能照常運作。
- **W5 bot 不受影響**：我的 W5 bot 部署在 Render，這支程式結束時會自動把 Webhook 換回 Render 的網址。

## 目前進度（2026-10-12）

- [x] Python 3.10、Node.js、Ollama 已安裝
- [x] cloudflared 已安裝（version 2026.10.0）
- [x] 下載 `gemma4:e4b`（6.6GB，完成）
- [ ] 執行 `line_bot_local.py`，接上 LINE
- [ ] 手機實測：問需要上網查的問題，確認回覆附來源
- [ ] 截圖：手機回覆＋同一時間的電腦終端機

## 安全注意

- 這個 repo 裡**沒有任何金鑰**（LINE 通行碼、Channel secret、試算表 ID 都是執行時才輸入）。
- 截圖時不要拍到通行碼、Channel secret。
- 訊息會經過 LINE 和 Cloudflare，搜尋會送到 Ollama 的線上服務；不要拿來處理別人的個人資料。
