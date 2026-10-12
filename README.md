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

## 怎麼裝

### macOS（我完成作業用的電腦：MacBook Pro M2）

我的 Mac 沒有 Homebrew，所以用下載的方式安裝（有 Homebrew 的話可以 `brew install ollama cloudflared`）：

```
python3 --version      # 要 3.10 以上（我的是 3.13.5）

# Ollama：從 https://ollama.com/download 下載 Mac 版並打開，再把指令連到 ~/bin（不需要管理員密碼）
mkdir -p ~/bin
ln -sf /Applications/Ollama.app/Contents/Resources/ollama ~/bin/ollama

# cloudflared：下載 Apple 晶片版
curl -L -o cloudflared.tgz https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-darwin-arm64.tgz
tar -xzf cloudflared.tgz && mv cloudflared ~/bin/ && rm cloudflared.tgz
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

ollama pull gemma4:e4b   # Mac 上是 9.5GB（Apple 晶片專用版）；記憶體 8GB 的 Mac 改用 gemma4:e2b
git clone https://github.com/yh839103-cmd/line-bot-local
cd line-bot-local
```

### Windows（PowerShell）

```
python --version
ollama pull gemma4:e4b
winget install --id Cloudflare.cloudflared -e --accept-package-agreements --accept-source-agreements
git clone https://github.com/yh839103-cmd/line-bot-local
cd line-bot-local
```

裝完 cloudflared 要**關掉終端機、開新的視窗**，不然程式會說找不到 cloudflared。

### LINE 設定

- LINE Developers →「Basic settings」→ **Channel secret**
- LINE Developers →「Messaging API」→ **Channel access token (long-lived)**（不要按 Reissue，會讓 W5 的 bot 失效）
- LINE Official Account Manager → 設定 → 回應設定：**Webhook 開啟、自動回應訊息關閉**

## 怎麼跑

先確認 Ollama 有開著（`ollama list` 看得到 `gemma4:e4b`），然後：

```
python3 line_bot_local.py      # Windows 用 python
```

1. 依序貼上 LINE 長效通行碼、Channel secret（畫面不會顯示），試算表 ID 直接按 Enter 跳過
2. 等畫面出現「開好了」，用手機傳訊息給 bot
3. 結束時按 **Ctrl+C**（不要直接關視窗），程式會把 Webhook 換回原本的網址

測試問題範例：「2026 年諾貝爾物理獎得主是誰？」→ 模型會先上網搜尋，回覆附來源。
小一點的模型：`MODEL=gemma4:e2b python3 line_bot_local.py`（Windows：先 `$env:MODEL="gemma4:e2b"`）

## 我改了什麼

1. **換 Webhook 失敗時自動重試**（`line_bot_local.py` 的 `main()`，有註解【我的修改】）
   - 問題：在學校電腦執行時，通道網址剛建立，LINE 回「`Invalid webhook endpoint URL`」，原版只試一次就放棄，Bot 沒接上。而且因為沒換成功，按 Ctrl+C 時也不會自動換回原本的網址。
   - 修改：遇到這個錯誤時每 5 秒重試，最多 8 次；成功後一樣會在 Ctrl+C 時換回原本的網址。
   - 測試：模擬 LINE 前兩次拒絕、第三次接受，確認程式重試後成功切換，結束時也換回原本的 Render 網址。
2. **沒有啟用 gws**（設定上的選擇）：照片存雲端硬碟、排日曆、寫試算表這三個功能沒有使用，只用「問答＋上網搜尋」。程式會顯示「找不到 gws.exe」的警告，但問答照常運作。
3. **W5 bot 不受影響**：我的 W5 bot 部署在 Render，程式結束時會自動把 Webhook 換回 Render 的網址。

## 卡住的地方與解法

| 狀況 | 原因 | 解法 |
|---|---|---|
| `can't open file ... line_bot_local.py` | 終端機不在程式所在的資料夾 | `git clone` 這個 repo，`cd line-bot-local` 再執行 |
| 找不到 cloudflared | 終端機是安裝 cloudflared 之前開的 | 關掉終端機、開新的視窗 |
| 換 Webhook 失敗 400：Invalid webhook endpoint URL | 通道網址剛建立，LINE 還查不到 | 程式改成自動重試（見「我改了什麼」） |
| 推送 GitHub 時 403 | 學校電腦存的是別的 GitHub 帳號 | 推送網址指定自己的帳號 |
| Mac 安裝 Ollama 指令要密碼，輸入失敗 | 安裝程式要把指令放進系統資料夾 | 改成把指令連到自己的 `~/bin`，不需要密碼 |
| 前兩題回「timed out」 | 模型剛載入，第一次推論比較慢（超過程式的 60 秒上限） | 再問一次就好，熱身後約 20 秒 |

## 實測結果（2026-10-12，MacBook Pro M2／24GB，gemma4:e4b）

| 時間 | 事件 | 結果 |
|---|---|---|
| 14:00:29–14:00:46 | 換 Webhook：LINE 回「通道網址還沒生效」 | 自動重試 4 次（**我的修改派上用場**） |
| 14:00:51 | 第 5 次換 Webhook | ✅ 成功，LINE 測試連線通了 |
| 14:02、14:03 | 手機問「2026 年諾貝爾物理獎得主是誰？」 | ❌ timed out：模型剛載入，前兩題超過 60 秒 |
| 14:04 | 同一題再問一次 | ✅ 模型自己決定上網搜尋，**23 秒**回覆並附來源 |
| 14:04 | 問「國立東華大學 陳文盛 老師」 | ✅ 上網搜尋後 **18 秒**回覆 |
| 結束 | 按 Ctrl+C | Webhook 換回原本的 Render 網址 |

觀察：模型剛載入時第一、二題比較慢，熱身後搜尋加回答約 20 秒。

## 進度

- [x] 在學校 Windows 電腦：安裝完成，連上 LINE、模型載入顯示卡（換 Webhook 時遇到 400 錯誤）
- [x] 修改程式：換 Webhook 失敗時自動重試
- [x] 在 Mac 上重新安裝並執行，成功接上 LINE
- [x] 手機實測：問需要上網查的問題，回覆附來源
- [x] 截圖：手機回覆＋同一時間的電腦終端機

## 安全注意

- 這個 repo 裡**沒有任何金鑰**（LINE 通行碼、Channel secret、試算表 ID 都是執行時才輸入）。
- 截圖時不要拍到通行碼、Channel secret。
- 訊息會經過 LINE 和 Cloudflare，搜尋會送到 Ollama 的線上服務；不要拿來處理別人的個人資料。
