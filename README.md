# OpenCode Browse：離線搬運包（可重現安裝 / 重現建置）

給沒有網路的內網環境用。裡面有三種東西，可以只用第一種，也可以全部帶著。

- `extension/`：已經建置好的擴充功能本體，直接「載入未封裝擴充功能」就能裝，不需要建置、不需要網路。
- `release/`：官方發佈的原始檔案（擴充功能 zip、各平台 relay 執行檔、雜湊檔）。要驗雜湊、或要在別的平台用時看這裡。
- `repo/`：上游完整原始碼，要在內網重新建置時用。

## 來源

- 上游專案：https://github.com/anomalyco/browse （MIT）
- 分支與提交：`dev`，`ae5d9d07ba8714bc34fa3ae5f5d899a0088e8782`（2026-11-20）
- 版本：`0.1.5`（對應 Chrome Web Store 上的 OpenCode Browse 0.1.5）
- 擴充功能 ID（商店版）：`mfnicocicmmlkpjnaffgihfjhdgjkdjg`

## 這個擴充功能不是單獨就能用

擴充功能本身只是通道。真正在做事的是本機的 relay 程式 `browse`（一個 MCP server），
擴充功能只跟 `127.0.0.1` 上的 relay 通訊，不連任何外部伺服器。

所以只裝擴充功能會卡在「未連線」。要一起裝 `browse` 執行檔。

## 最快路線：Windows 內網機器

1. 解開 relay 執行檔：

   ```powershell
   tar -xzf release/cli-windows-x64.tgz
   mkdir "$env:USERPROFILE\.local\bin" -Force
   copy package\bin\browse.exe "$env:USERPROFILE\.local\bin\browse.exe"
   ```

   如果這台機器很舊（CPU 沒有 AVX2）導致 `browse.exe` 一啟動就 crash，才需要 baseline 版；
   我沒帶進來，要從 `https://anoma.ly/release/browse/cli/stable.json` 的 `windows-x64-baseline` 另外抓（要有網路）。

2. 關掉自動更新（內網連不到更新伺服器，關掉才不會每次卡住）：

   ```powershell
   browse update --mode off
   ```

3. 裝擴充功能：
   - 開 `chrome://extensions`
   - 右上角開「開發人員模式」
   - 按「載入未封裝項目」，選這個資料夾裡的 `extension/`
   - 建議把 `extension/` 複製到一個固定的本機路徑再載入（例如 `%USERPROFILE%\.local\share\browse-extension`），
     不要從網路磁碟或會被清掉的位置載入

4. 配對（讓 relay 認得這個瀏覽器）：

   ```powershell
   browse pair
   ```

   它會印一組一次性配對碼，並等瀏覽器來領。把那段配對碼貼到擴充功能的設定頁（`chrome://extensions`
   → OpenCode Browse → 擴充功能選項），或直接開它印出來的 `127.0.0.1` 網址。

5. 確認：

   ```powershell
   browse status
   browse navigate https://example.com
   ```

   `browse status` 要看到 relay 在跑、瀏覽器已配對。`browse navigate` 會開一個背景分頁並印出 tab id。

6. 接到 agent（可選）：`browse install --no-open -a opencode` 會把 `browse mcp` 寫進 opencode 的 MCP 設定，
   不用自己去改 config。其他 agent 換掉 `-a` 的名字即可。

macOS / Linux 的步驟一樣，只是執行檔換成 `release/cli-macos-arm64.tgz` 或 `release/cli-linux-x64.tgz`，
放到 `~/.local/bin/browse`（或 `$XDG_BIN_HOME`）。

## 這個包裡的東西

`extension/` 的檔案跟 Chrome Web Store 上 0.1.5 的內容一致，是從官方 release zip
（`release/browse-extension-0.1.5.zip`）原樣解開的。manifest 裡沒有 `key` 和 `update_url`，
所以用「載入未封裝」安裝時，Chrome 會給它一個新的擴充功能 ID（跟商店版的 ID 不同）。
這是正常的，配對是靠本機 relay 的一次性配對碼，不看 ID。

`release/` 內含：

- `browse-extension-0.1.5.zip`：擴充功能建置產物（= `extension/`）
- `opencode-browse-0.1.5.tgz`：npm 上 `@opencode/browse` 的啟動器（會連網下載原生執行檔，內網不要用這個）
- `cli-windows-x64.tgz`、`cli-macos-arm64.tgz`、`cli-linux-x64.tgz`：各平台 relay 原生執行檔
- `SHA256SUMS`：GitHub release 附的雜湊（只涵蓋上面兩個 GitHub asset，不含 `cli-*.tgz`）

`cli-*.tgz` 的 sha256 對照（來自 anoma.ly release manifest，版本 0.1.5）：

```text
windows-x64     05f0e747e2d0d099aefec37caf6605d2464812a047ad14c1e2952e8b6e83991a
macos-arm64     d63b30d9b89d05070a3310d2d80435e06ab9cc35216b801acc68f969e14202bf
linux-x64       8ba73f16322a42544cad3a97490bfd14957b97b108b58f43913e6ca088a3cdd7
```

驗法（PowerShell）：`Get-FileHash release\cli-windows-x64.tgz -Algorithm SHA256`。

## 要從原始碼重新建置

`repo/` 是上游完整原始碼（已移除 `.git`，你自己 `git init` 就好）。

需求：`bun` 1.4.4，**版本要對**（`package.json` 的 `packageManager` 指定的）。用舊版會直接失敗：

```text
error: Unknown lockfile version   (bun 1.3.9 讀 bun.lock v2 時)
```

```bash
cd repo
bun install          # 這步需要套件來源，約 330 個套件，317 MB
bun run build        # 產出 dist/，就是擴充功能本體
```

`bun run build` 會跑兩次 vite：一次建 background 與三個頁面，一次單獨建 content script，
結果都在 `repo/dist`。這個 `dist/` 可以直接「載入未封裝」。

如果還要建 relay CLI：

```bash
cd repo/cli
bun install
bun run build        # 產出 cli/dist/cli.mjs
```

### 內網建置的現實問題

`bun install` 一定要有套件來源。內網三種做法，由簡到難：

1. 內網有 npm mirror（Artifactory / Nexus 之類）時，設 `~/.npmrc` 指過去即可，`bun install` 會讀。
2. 沒有 mirror：在有網路的機器上跑 `bun install`，把整個 `node_modules/` 連同 `bun.lock` 一起複製進內網，
   之後建置不必再安裝。`node_modules` 很大（數百 MB），不要放進 git，用實體媒介或內部檔案伺服器搬。
3. 直接跳過建置：用 `extension/`。這是同樣的程式碼、同樣的 0.1.5，只差沒有原始碼。

`bun install` 和 `bun run build` 在本機（macOS arm64、bun 1.4.4）實測都成功，
`dist/` 的檔案佈局與 `extension/` 一致（檔名裡的 hash 因建置時間戳與工具鏈版本略有不同，功能相同）。

## 企業環境可能擋住的地方

- **開發人員模式被政策關掉**：`chrome://extensions` 載入未封裝會被拒。要請 IT 用企業政策放行，
  或用 `ExtensionInstallForcelist` / `ExtensionSettings` 派送 self-hosted CRX。
- **`debugger` 權限**：擴充功能靠 `chrome.debugger`（CDP）驅動分頁。若政策禁 `debugger` 權限，功能會壞，
  這個沒有替代方案。
- **self-hosted CRX**：需要自己簽（`chrome --pack-extension`），並在 manifest 加上 `update_url`
  指向內部的 update manifest，再由政策派送。純「載入未封裝」在企業機器上通常就夠了，先試這條。
- **relay 的埠**：只聽 `127.0.0.1`。若瀏覽器在另一台機器，要 `browse set hostname 0.0.0.0` 並自行處理防火牆。

## 之後怎麼更新

內網環境不建議開自動更新（連不到）。要換版本時，重複本流程：抓新的 release zip 與對應平台 CLI、
換掉 `extension/` 與 `browse.exe`、重開瀏覽器、`browse status` 確認。

## 我沒驗證的部分

- 沒有在 Windows 上實際跑過 `browse.exe` 與 `chrome://extensions` 載入流程（這台是 macOS arm64）。
- 沒有實測企業政策的限制（開發人員模式、`debugger` 權限）在 TSMC 機器上的實際狀況。
- 沒有實測「載入未封裝」後 `browse pair` 的配對流程（本機已配對，沒有重跑）。
- `cli-macos-arm64.tgz` / `cli-linux-x64.tgz` 只下載沒有執行。
