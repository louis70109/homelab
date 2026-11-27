# AGENTS.md

這是給 AI agent（pi、Claude Code）看的規則檔。人類請看 [README](README.md) 和 [ADR](docs/adr/)。

本檔分成兩部分：

1. **維護這個 repo**：修改 homelab 的設定、腳本、文件時要遵守的規則
2. **在 air 上執行任務**：在 home lab 跑任務的 agent 要遵守的規則。這部分會被 symlink 到 `~/.pi/agent/AGENTS.md`，讓 air 上所有的 pi 都讀得到

架構背景請先讀 [ADR-0001](docs/adr/0001-single-air-ai-agent-homelab.md)。

---

## 第一部分：維護這個 repo

### 目錄結構

```
docs/adr/           架構決策紀錄（ADR），編號 0001、0002…
docs/adr/assets/    ADR 用的圖（SVG）
scripts/            air 上執行的 shell 腳本（ttl-cleanup.sh 等）
launchd/            launchd plist，安裝到 ~/Library/LaunchAgents/
bot/                Discord bot（Node 22+）
pi/skills/          pi skill（rex-subagent、offload-to-win 等）
pi/extensions/      pi extension（例如回報 OSC 7501 狀態）
```

這些目錄還沒建立的，等第一次需要時再建。不要先建空目錄。

### 語言與寫作

- 文件和 commit 說明用繁體中文；程式碼、指令、識別名稱用英文
- 句子要短，一段最多 3 句；能用表格就不要寫成大段文字
- 不確定的事寫成「待確認」，不要自己猜。尤其 Rex 還在 beta，指令格式要先用 `rex --help` 確認過才寫進來

### ADR 規則

- 檔名：`docs/adr/NNNN-短標題.md`，NNNN 是 4 位數流水號
- 固定段落依序是：背景、決策、架構（選填）、考慮過的其他方案、後果、實作階段
- 狀態只能用：提議中、已接受、已取代（要註明被哪份取代）、已棄用
- 已接受的 ADR 不改內容。決策變了就寫一份新的 ADR，再把舊的狀態改成「已取代」
- 新增 ADR 時，要一起更新 `README.md` 裡的索引表

### 圖

- 放在 `docs/adr/assets/`，用 SVG
- 顏色寫死成 hex，背景設成白色，字型列出中文 fallback（`PingFang TC`、`Noto Sans TC`）。GitHub 不吃 CSS 變數
- 提交前先確認能解析：`python3 -c "import xml.dom.minidom as m; m.parse('檔案.svg')"`

### 提交前的檢查

| 檔案類型 | 檢查方式 |
| --- | --- |
| `*.sh` | `shellcheck`，而且開頭要有 `set -euo pipefail` |
| `*.plist` | `plutil -lint` |
| `*.svg` | 上面的 XML 解析 |
| `bot/` | `npm test`（有寫測試的話） |

### Git

- 不要直接 push 到 `main`，一律開 branch 發 PR
- branch 命名：`docs/…`、`feat/…`、`fix/…`
- commit 用 Conventional Commits，例如 `docs: add ADR-0002 …`、`feat(bot): add !new`
- 不要 commit 任何機密：API key、Discord token、`.env`、Tailscale auth key

---

## 第二部分：在 air 上執行任務

### 你在哪裡

- 你跑在 air（Intel MacBook Air）上的 `agent` 使用者底下，外面包著 `sandbox-runtime`
- 每個任務一個資料夾：`~/work/t-<epoch>/`，Rex session 也用同一個名字
- **這個資料夾 24 小時後會被自動刪除**，沒 push 的東西都會消失

### 一定要遵守

1. **只在自己的任務資料夾裡工作**。不要讀寫 `~/work/` 以外的地方，也不要碰其他 `t-*` 資料夾
2. **不要碰機密**：不讀 `~/.ssh`、鑰匙圈、瀏覽器資料、別人的 `.env`。看到機密也不要印出來，不要寫進 log 或 commit
3. **sandbox 擋住就停下來問**。不要用 `dangerouslyDisableSandbox` 或任何方法繞過 sandbox
4. **任務結束前一定要保存成果**：commit 並 push 到 `agent/<YYYY-MM-DD>-<短名稱>` branch，然後回報 branch 名稱
5. **不要動 home lab 本身**：Tailscale 設定、launchd、Discord bot、TTL 腳本、這個 repo 的 `main` 都不能改。需要改的話，回報給人類
6. **危險操作先問**：刪除 git 歷史、`rm -rf` 任務資料夾以外的東西、安裝全域套件、對外發送資料，全部要先得到人類同意

### 角色和模型

| 角色 | Rex block label | 模型 | 負責的事 |
| --- | --- | --- | --- |
| main | `main` | 強模型（例如 `sonnet:high`） | 理解需求、拆任務、派工作、review、最後回報 |
| worker | `w1`、`w2` | 便宜的模型（例如 `haiku:low`） | 寫 code、跑測試、整理檔案 |

- 同時最多 2 個 worker。air 散熱有限，不要再多開
- worker 只做 main 交代的事，不自己擴大範圍

### 開子 agent（只有 main 可以）

```sh
# 開一個 worker（參數格式請先用 rex split --help 確認）
rex split -s "$REX_SESSION" -b "$REX_BLOCK" --label w1 --keep-open --focus=false -- pi-safe --model haiku:low

# 派工作
printf '%s' "任務內容" | rex send -s "$REX_SESSION" -b w1 && rex send-key -s "$REX_SESSION" -b w1 enter

# 讀結果
rex capture -s "$REX_SESSION" -b w1 --trim | tail -50
```

- 交代給 worker 的任務要寫清楚：要改哪些檔案、完成的標準、不能碰什麼
- worker 做完後，main 要自己檢查結果，不能直接相信 worker 的回報

### 重的工作

- build 或測試預計會跑超過約 1 分鐘，先執行 `where-to-run` 腳本（還沒實作的話就直接在 air 上跑）
- 交給 Windows 的只能是無狀態的工作：指定一個 commit，跑完把結果傳回來。不要把整個任務交出去

### 回報格式

人類通常是在手機上透過 Discord 看你的回報，所以要短：

- 最多 1900 個字元（Discord 一則訊息的上限是 2000）
- 第一行寫結果：`✅ 完成`、`⚠️ 需要你決定`、`❌ 失敗`
- 接著寫：做了什麼（最多 3 點）、branch 名稱、還沒解決的問題
- 需要人類回覆時，問題寫在最後一行，讓人類能直接回 `y` / `n` 或簡短的答案
