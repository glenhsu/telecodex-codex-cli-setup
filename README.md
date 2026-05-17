# TeleCodex + Codex CLI 整合設定指南

> 讓你可以用 Telegram 遙控 Codex CLI 的完整踩坑記錄

TeleCodex 是一個 Telegram ↔ Codex CLI 的橋接工具，讓你用手機就能跟 Codex 對話、發送語音、圖片、文件。但要把 TeleCodex 接上**使用自訂 provider（例如 oMLX）的 Codex CLI**，中間有不少坑要踩。

這份指南記錄了從零到可用的完整過程。

## 目錄

- [前置需求](#前置需求)
- [基本安裝](#基本安裝)
- [穩定位置與自動啟動](#穩定位置與自動啟動)
- [步驟一：設定 .env](#步驟一設定-env)
- [步驟二：連接 Codex CLI 的自訂 Provider](#步驟二連接-codex-cli-的自訂-provider)
- [步驟三：繞過 macOS Gatekeeper](#步驟三繞過-macos-gatekeeper)
- [步驟四：啟動與測試](#步驟四啟動與測試)
- [踩坑記錄](#踩坑記錄)
- [完整範例 .env](#完整範例-env)
- [給原作者的 PR 建議](#給原作者的-pr-建議)

## 前置需求

- macOS（Apple Silicon 或 Intel 均可）
- Node.js 22+
- 已安裝並設定好 Codex CLI（`codex` 在 PATH 上）
- Telegram bot token（跟 [@BotFather](https://t.me/botFather) 申請）
- 你的 Telegram User ID（找 [@userinfobot](https://t.me/userinfobot)）

## 基本安裝

```bash
# Clone 專案到穩定位置（不要 clone 到 /tmp/）
git clone https://github.com/benedict2310/telecodex.git ~/telecodex
cd ~/telecodex

# 安裝依賴
npm install

# 複製環境變數範本
cp .env.example .env
```

> **重要：** 專案位置請放在 `~/telecodex/` 或家目錄下的穩定位置。不要放在 `/tmp/`，重開機後檔案會消失。

## 穩定位置與自動啟動

TeleCodex 必須放在不會被重開機清掉的位置。推薦 `~/telecodex/`。

### LaunchAgent 自動啟動（開機即跑）

建立 `~/Library/LaunchAgents/com.telecodex.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.telecodex</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/athing/telecodex/run.sh</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/athing/telecodex</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/athing/telecodex/telecodex.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/athing/telecodex/telecodex.err</string>
    <key>ThrottleInterval</key>
    <integer>5</integer>
</dict>
</plist>
```

### run.sh（LaunchAgent entrypoint）

建立 `~/telecodex/run.sh`：

```bash
#!/bin/bash
export PATH="/opt/homebrew/bin:$PATH"
export OMLX_API_KEY=apiapi
cd /Users/athing/telecodex
exec npx tsx src/index.ts
```

### 生命週期管理

```bash
# 載入（啟動）
launchctl load ~/Library/LaunchAgents/com.telecodex.plist

# 卸載（停止）
launchctl unload ~/Library/LaunchAgents/com.telecodex.plist

# 檢查狀態
launchctl list | grep telecodex
# → PID  STATUS  com.telecodex
#   STATUS 0 = 正常, 127 = crash

# 查看日誌
cat ~/telecodex/telecodex.log
cat ~/telecodex/telecodex.err
```

### codex wrapper（codex CLI 時自動啟動 telecodex）

建立 `~/.local/bin/codex`，確保 PATH 優先於 `/opt/homebrew/bin/codex`：

```bash
#!/bin/bash
TELECODEX_DIR="$HOME/telecodex"

if ! pgrep -f "telecodex" >/dev/null 2>&1; then
    echo "📱 Starting telecodex bridge..."
    cd "$TELECODEX_DIR" && OMLX_API_KEY=apiapi npx tsx src/index.ts &
    sleep 2
fi

exec /opt/homebrew/bin/codex "$@"
```

運作邏輯：
1. 檢查 `pgrep -f telecodex` — LaunchAgent 已在跑就不重複啟動
2. 如果 telecodex 沒在跑（LaunchAgent 被關掉或 crash），自動啟動它
3. 然後 `exec` 真正的 codex binary

## 步驟一：設定 .env

編輯 `.env`，填入必要資訊：

```env
# Telegram (必要)
TELEGRAM_BOT_TOKEN=你的bot_token
TELEGRAM_ALLOWED_USER_IDS=你的user_id

# Codex API Key (如果只用登入可以省略)
# CODEX_API_KEY=你的codex_api_key
```

其他選項可以先保持預設。

## 步驟二：連接 Codex CLI 的自訂 Provider

> **核心坑點**：TeleCodex 透過 `@openai/codex-sdk` 啟動 Codex 子程序，這個 SDK 會讀取 Codex CLI 的設定檔 `~/.codex/config.toml` 來決定用哪個 provider 和 model。

如果你的 Codex CLI 使用了自訂 provider（例如本地的 oMLX、NVIDIA NIM 等），**SDK 會自動繼承這些設定**，只要環境變數有設對就行。

### Codex CLI config 範例

```toml
# ~/.codex/config.toml
model_provider = "omlx"
model = "Qwen3.6-35B-A3B-UD-MLX-4bit"

[model_providers.omlx]
name = "oMLX"
base_url = "http://127.0.0.1:8989/v1"
env_key = "OMLX_API_KEY"
```

### 關鍵：對應的環境變數

當 SDK 啟動 Codex 子程序時，它會讀取 provider 所需的環境變數。以上面的設定來說，需要設定：

```env
# .env (給 telecodex 的)
OMLX_API_KEY=apiapi
CODEX_API_KEY=apiapi
```

> **為什麼要兩個？**
> - `OMLX_API_KEY` — Codex CLI 內部用這個 key 去跟 oMLX server 驗證
> - `CODEX_API_KEY` — TeleCodex 用這個 key 初始化 Codex SDK

> oMLX 的 API key 可以在 `~/.omlx/settings.json` 的 `auth.api_key` 找到。

> 如果你用的是 NVIDIA NIM 或其他遠端 provider，env_key 對應的環境變數也要跟著設。

## 步驟三：繞過 macOS Gatekeeper

### 症狀

啟動 telecodex 後，bot 有回應但一直報錯：

```
Error: Failed to spawn Codex CLI: EACCES
```

或是 macOS 彈出「無法打開 codex，因為 Apple 無法檢查其是否包含惡意軟體」。

或是在 `系統設定 > 隱私權與安全性` 完全看不到提示。

### 原因

`@openai/codex-sdk` 在 `npm install` 時會下載一個內建的 codex binary 到 `node_modules/@openai/codex-darwin-arm64/vendor/aarch64-apple-darwin/codex/`。這個 binary 沒有經過 Apple 公證，第一次被執行時 macOS 會用 **Gatekeeper 把它直接隔離刪除**，不會留任何 UI 提示讓你手動允許。

Binary 被刪掉後，SDK 找不到執行檔就直接報錯。

### 解法 A：讓 SDK 指向本機安裝的 codex

修改 `src/codex-session.ts`，在 `resetCodexClient()` 中傳入 `codexPathOverride`：

```typescript
// src/codex-session.ts
private resetCodexClient(): void {
    this.codex = new Codex({
      codexPathOverride: '/opt/homebrew/bin/codex',  // ← 加上這行
      apiKey: this.config.codexApiKey,
      config: {
        approval_policy: this.currentLaunchProfile.approvalPolicy,
      },
      env: buildCodexEnv(this.config.codexApiKey),
    });
}
```

> 這個參數直接對應 SDK 內部 `CodexExec` 建構式的 `executablePath`：
> ```javascript
> constructor(executablePath = null, env, configOverrides) {
>     this.executablePath = executablePath || findCodexPath();
> }
> ```

### 解法 B：手動重建並允許 binary

如果你不想改程式碼，也可以手動把 binary 放回去：

```bash
# 重新安裝讓 SDK 重新下載 binary
npm install

# 找到 binary 位置
find node_modules/@openai/codex-darwin-arm64 -type f -name "codex*"
# → vendor/aarch64-apple-darwin/codex/

# 手動允許執行
xattr -dr com.apple.quarantine node_modules/@openai/codex-darwin-arm64/
```

但 macOS 下次還是可能會再刪掉。**解法 A 是真正一勞永逸的方式。**

## 步驟四：啟動與測試

### 手動啟動

```bash
cd ~/telecodex && npx tsx src/index.ts
```

### 透過 LaunchAgent 啟動（開機即跑）

```bash
launchctl load ~/Library/LaunchAgents/com.telecodex.plist
```

成功啟動時會看到：

```
TeleCodex running
Auth: authenticated (api-key)
Workspace: /path/to/telecodex
Default launch profile: Default (workspace-write / never)
Session mode: per Telegram context
```

然後去 Telegram 對你的 bot 發訊息，應該就會開始回應了。

## 踩坑記錄

### ❌ 「Missing environment variable: OMLX_API_KEY」

**原因：** Codex CLI config 指定了 `env_key = "OMLX_API_KEY"`，但 `.env` 裡沒設這個變數。

**解法：** 在 `.env` 中補上對應 provider 的 API key 環境變數。查看你的 `~/.codex/config.toml` 找到 `model_providers.<name>.env_key`，填入對應值。

如果 key 是 localhost 不需要驗證的 server，填入 `not-needed-localhost` 通常也能繞過檢查。

### ❌ 「Failed to spawn Codex CLI: EACCES」

**原因：** SDK 內建的 binary 被 macOS Gatekeeper 刪除了。

**解法：** 見[步驟三解法 A](#解法-a讓-sdk-指向本機安裝的-codex)。

### ❌ macOS Gatekeeper 連 UI 提示都沒有

**原因：** macOS Sequoia 的 Gatekeeper 在某些情況下會直接靜默隔離並刪除未公證的執行檔，不彈出對話框，也不會出現在「隱私權與安全性」設定中。

**解法：** 只能從源頭解決 — 不要讓 SDK 去執行內建 binary，改用本機已安裝的 codex。

### ❌ Bot 連線成功但指令無回應

**原因：** 通常是 API key 或 model provider 設定不匹配。

**檢查清單：**
1. Codex CLI 本身能不能正常運作？`codex --version`
2. Provider server 有沒有在跑？`ps aux | grep omlx`
3. `.env` 有沒有設對對應的環境變數？
4. 查看 TeleCodex 啟動 log，確認有沒有「Auth: authenticated」和「Workspace: ...」

### ❌ 「TeleCodex is running but Codex is still initializing...」

這是正常的。第一次啟動時 Codex 需要載入模型（對於本地 LLM 可能需要幾十秒到幾分鐘），等 loading 完成就好了。

### ❌ LaunchAgent 無法啟動（node: No such file or directory）

**原因：** launchd 的環境 PATH 只有 `/usr/bin:/bin:/usr/sbin:/sbin`，不含 `/opt/homebrew/bin`。直接在 plist 的 ProgramArguments 寫 `npx tsx` 會找不到 node。

**解法：** 透過 wrapper script (`run.sh`) 執行，在 script 內先設定 `PATH="/opt/homebrew/bin:$PATH"`。

### ❌ codex wrapper 用 alias 卻沒生效

**原因：** alias 在 non-interactive shell 或 script 中不會生效。

**解法：** 用 `~/.local/bin/codex` 放在 PATH 前面比 alias 更可靠。確保 `~/.zshrc` 有 `export PATH="$HOME/.local/bin:$PATH"`。

### ❌ 409 Conflict on telecodex restart

**原因：** 舊的 `getUpdates` polling 還持有 bot token 連線。

**解法：** Kill 所有舊 process 再重啟：
```bash
ps aux | grep tsx | grep -v grep | awk '{print $2}' | xargs kill
sleep 10
launchctl load ~/Library/LaunchAgents/com.telecodex.plist
```

### ❌ 搬遷後 workspace 路徑問題

**原因：** 從 `/tmp/telecodex` 搬到 `~/telecodex` 後，telecodex 的 runtime state（`~/.telecodex/contexts.json`）會自動記錄新的 workspace 路徑，重新啟動後即可。不需要手動處理。

## 完整範例 .env

這是以 oMLX 為 provider 的完整 `.env` 範例：

```env
# Telegram
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNO-pqrSTUvwxYZ
TELEGRAM_ALLOWED_USER_IDS=987654321

# Codex CLI (用 oMLX local provider)
OMLX_API_KEY=apiapi
CODEX_API_KEY=apiapi
CODEX_MODEL=Qwen3.6-35B-A3B-UD-MLX-4bit

# Codex binary 路徑（配合 codexPathOverride）
CODEX_BINARY_PATH=/opt/homebrew/bin/codex

# Sandbox & approval (安全起見從 workspace-write 開始)
CODEX_SANDBOX_MODE=workspace-write
CODEX_APPROVAL_POLICY=never

# 其他選項
MAX_FILE_SIZE=20971520
TOOL_VERBOSITY=summary
SHOW_TURN_TOKEN_USAGE=false
ENABLE_TELEGRAM_LOGIN=true
ENABLE_TELEGRAM_REACTIONS=false
```

## 給原作者的 PR 建議

原始 TeleCodex 專案需要增加的兩個東西：

1. **`CODEX_PATH` 環境變數支援** — 在 `.env.example` 中新增 `CODEX_PATH`，讓使用者在 codex binary 路徑有問題時可以直接指定，不用改原始碼
2. **Troubleshooting 章節** — 在 README 中加入上述踩坑記錄，幫助遇到同樣問題的使用者

兩者的實作已經包含在這份指南中。

---

*如果有遇到其他坑，歡迎開 Issue 或 PR 補充！*
