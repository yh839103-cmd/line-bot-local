# LINE bot 接本地開源模型（W6、W7 作業）

| 程式 | 作業 | 做什麼 |
|---|---|---|
| [`apps-script/Code.gs`](apps-script/Code.gs)＋[`apps-script/line-search-relay.mjs`](apps-script/line-search-relay.mjs) | **W7：Apps Script 當 Webhook** | bot 留在 Apps Script（24 小時開著），「查 …」轉回我電腦的本地模型 |
| [`line_bot_lite.py`](line_bot_lite.py) | W6 回家作業：輕量版 | 只回答文字問題；有「模擬模式」，不接 LINE 也能在終端機測試 |
| [`line_bot_local.py`](line_bot_local.py) | W6 課堂作業 | 接真的 LINE，模型會上網搜尋、附來源 |

---

## W7：Apps Script 當 Webhook，「查 …」轉給我電腦的本地模型

```
手機 LINE → LINE 平台 → Apps Script（Google 機房，一直開著）
                          ├─ 一般訊息：Gemini 整理成待辦 → 寫進試算表、排 Google 日曆
                          └─「查 …」：Cloudflare 通道 → 我電腦上的轉接站 line-search-relay.mjs
                                                        → 本地模型 gemma4:e4b 上網搜尋 → 答案＋來源原路回到手機
```

電腦關掉時，只有「查 …」會回「🔌 本地模型現在連不到」，記待辦照常運作。

### 檔案

| 檔案 | 說明 |
|---|---|
| `apps-script/Code.gs` | 貼進 Apps Script 的程式（**改後**） |
| `apps-script/Code.before.gs` | 改前的版本（老師的 Drive 版對照答案） |
| `apps-script/Code.diff` | 改前／改後對照 |
| `apps-script/appsscript.json` | Apps Script 資訊清單（權限設定） |
| `apps-script/line-search-relay.mjs` | 跑在我電腦上的轉接站（課程提供，沒改） |

### 怎麼裝（Windows PowerShell，一次貼一行）

```
winget install --id Ollama.Ollama -e --accept-package-agreements --accept-source-agreements --silent
$env:PATH += ";$env:LOCALAPPDATA\Programs\Ollama"
ollama pull gemma4:e4b
winget install --id OpenJS.NodeJS.LTS -e --accept-package-agreements --accept-source-agreements --silent
$env:PATH += ";$env:ProgramFiles\nodejs;$env:APPDATA\npm"
New-Item -ItemType Directory -Force "$env:USERPROFILE\line-relay" | Out-Null; Set-Location "$env:USERPROFILE\line-relay"
Invoke-WebRequest -Uri https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe -OutFile cloudflared.exe
$env:PATH += ";$env:USERPROFILE\line-relay"
```

再把 `apps-script/line-search-relay.mjs` 放進 `line-relay` 資料夾。

**Apps Script 那一端：**
1. 新建 Google 試算表 → 擴充功能 → Apps Script，貼上 `Code.gs`；專案設定勾「顯示 appsscript.json」，貼上 `appsscript.json`。
2. 專案設定 → 指令碼屬性：`GEMINI_API_KEY`、`LINE_CHANNEL_ACCESS_TOKEN`。
3. 執行 `testSetup`、授權，確認 4 行 ✅。
4. 部署 → 網頁應用程式（執行身分：我；存取：所有人）→ 把 `/exec` 網址貼到 LINE Developers 的 Webhook URL。

### 怎麼跑

1. 開 Ollama，在 `line-relay` 資料夾執行 `node line-search-relay.mjs`。
2. 它會印出 `SEARCH_RELAY_URL`，密碼自動放進剪貼簿。把這兩個值填進 Apps Script 的指令碼屬性 `SEARCH_RELAY_URL`、`SEARCH_RELAY_KEY`。網址每次重開都會換，換了只要改屬性，不用重新部署。
3. 手機傳 `查 2026 年諾貝爾物理獎得主是誰？`（「查」後面要有空格）。
4. 結束時在轉接站視窗按 **Ctrl+C**。

### 我改了什麼

在老師的 Drive 版 `Code.gs` 上，請 AI 只加「查 …」這一條路，其他都不動（完整對照見 `apps-script/Code.diff`）：

1. **設定區**：新增 `SEARCH_RELAY_URL`、`SEARCH_RELAY_KEY`，都從指令碼屬性讀，**程式裡沒有網址和密碼**；另外新增 `SEARCH_PREFIX`，只有「查」或「搜尋」後面接空格或冒號才算查詢，「查一下明天…」照舊當待辦。
2. **`handleEvent`**：符合「查 …」就交給 `askLocalSearch`，不問 Gemini、不寫試算表。
3. **新增 `askLocalSearch`**：POST 到「網址/ask」，密碼放在 `x-relay-key` 標頭，最多等 50 秒。回覆附來源；連不到、密碼錯、忙線都老實回一句話，**不改問 Gemini**。
4. **退回 AI 一次**：第一版查詢時的「輸入中」動畫只有 20 秒，但本地模型上網查常要 20–40 秒，動畫會先消失、看起來像沒反應。我請 AI 改成 60 秒、其他不要動；它把 `showLoading` 加了秒數參數，待辦和照片仍是 20 秒。

### 卡住的地方

| 狀況 | 原因 | 解法 |
|---|---|---|
| PowerShell 報 `Unexpected token 'winget'` | 一次貼多行，兩行黏成一行 | 一次只貼一行 |
| appsscript.json 一直報錯 | 貼了好幾次，內容疊在一起 | Ctrl+A、Delete 清空再貼一次 |
| 執行選單只有 `myFunction`，找不到 testSetup | 程式被貼進原本的 `function myFunction() {` 裡面 | 全選刪掉再貼 |
| 執行後只有「執行完畢」，沒有 ✅ | 選單選到 `doPost`，不是 `testSetup` | 下拉選單改選 testSetup |
| 「查 …」被當成待辦（Gemini 回「似乎是查詢問題」） | 改完沒有發布新版本，LINE 打的還是舊版 | 管理部署作業 → 編輯 → 版本選「新版本」 |
| 密碼貼不上 | 截圖會蓋掉剪貼簿裡的密碼 | 用 `Get-Content "$env:USERPROFILE\.line-search-relay.key" \| Set-Clipboard` 再放一次，貼完才截圖 |

### 實測結果（2026-10-19，學校 Windows 電腦，模型在顯示卡上）

| 時間 | 事件 | 結果 |
|---|---|---|
| 10:09:05 | 轉接站載入 gemma4:e4b | ✅ 7 秒，模型在顯示卡上 |
| 10:31 | Apps Script `testSetup` | ✅ LINE token、Google 日曆、Gemini、試算表 4 項通過 |
| 10:3x | 手機傳「明天下午三點要開會」 | ✅ 記進試算表、日曆已建立 |
| 10:48:51 | 手機傳「查 2026年諾貝爾得獎主是誰」 | ✅ 模型自己搜尋，**5 秒**回答好，答 Francis Halzen（物理獎）並附 nobelprize.org 來源 |

### 安全

- 程式和截圖裡**沒有**長效通行碼、Channel secret、`/exec` 網址、通道網址和轉接站密碼；這些只放在 Apps Script 的指令碼屬性。
- 轉接站只聽本機 `127.0.0.1:8787`，外面只能從通道進來，而且要帶對密碼。

---

## W6 的兩支程式

| 程式 | 作業 | 做什麼 |
|---|---|---|
| [`line_bot_lite.py`](line_bot_lite.py) | **W6 回家作業：輕量版** | 只回答文字問題；有「模擬模式」，不接 LINE 也能在終端機測試 |
| [`line_bot_local.py`](line_bot_local.py) | W6 課堂作業 | 接真的 LINE，模型會上網搜尋、附來源 |

---

## W6 回家作業：LINE bot 輕量版（`line_bot_lite.py`）

### 怎麼裝

- Python 3（只用內建功能，不用 `pip install`）
- Ollama，並拉一個模型：
  ```
  ollama pull gemma4:e2b     # 程式預設的模型（4.6 GB），沒有顯示卡也跑得動
  ```
  我的 Mac 上週已經裝好 `gemma4:e4b`，所以直接用它，不用再下載。
- 下載這個 repo：`git clone https://github.com/yh839103-cmd/line-bot-local`，再 `cd line-bot-local`

### 怎麼跑

- **關卡 A：模擬模式**（不接 LINE、不用任何金鑰）
  ```
  export MODEL=gemma4:e4b             # 換成我已經有的模型（Windows：$env:MODEL="gemma4:e4b"）
  python3 line_bot_lite.py --simulate
  ```
  看到「📱 你：」就可以打字問問題，打 `quit` 結束。
- 選做，讓它上網查：`python3 line_bot_lite.py --simulate --search`（要先 `ollama signin`）
- 關卡 B，真的接 LINE：`python3 line_bot_lite.py`
  - 需要 cloudflared、LINE 長效通行碼和 Channel secret
  - 金鑰在開程式時才貼上，**不存進檔案**
  - 結束時按 **Ctrl+C**，Webhook 會換回原本的網址

### 我改了什麼、跑起來怎樣

- 模型：`gemma4:e4b`。程式預設是 `e2b`，我用環境變數 `MODEL` 換成電腦上已經有的 `e4b`，程式本身沒改。
- 電腦：MacBook Pro M2、記憶體 24 GB。沒有獨立顯示卡，Ollama 用 Apple 晶片內建的 GPU。
- 一題約 **2 秒**（2026-10-15 關卡 A 實測；第一次載入模型 8 秒）。上週課堂版上網搜尋加回答約 18–23 秒。
- 實測回答：
  - 「台灣最高的山」答玉山，但高度只說「約三千公尺」，實際是 3952 公尺，小模型記得不精確。
  - 「2026 諾貝爾物理獎」回答沒有上網、無法提供，沒有亂編，符合預期。
- 改了：`line_bot_lite.py` 沒改。課堂版 `line_bot_local.py` 改成換 Webhook 失敗時自動重試，說明在下面。
- 卡在：模型剛載入時第一題比較慢，課堂版前兩題 timed out，熱身後就正常。

---

## W6 課堂作業：接真 LINE、會上網查（`line_bot_local.py`）

讓 LINE bot 用**跑在自己電腦上的開源模型**（Ollama 的 Gemma 4，`gemma4:e4b`）回答問題。
需要最新資訊時，模型會自己上網搜尋，回覆最後附上來源網址。

課堂版的 bot 只有一支 Python 程式：[`line_bot_local.py`](line_bot_local.py)，是課程提供的版本，加上我的修改。

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
