+++
title = "用 Slack + Claude + Cloudflare Workers 做一個會自己長大的客服小幫手"
date = 2026-11-06T16:00:00+08:00
draft = false
description = "客服一直問重複的問題，我把舊的 QA 整理成知識庫，讓 Claude 在 Slack 上回答，答不出來就找工程師，工程師的回答再存回知識庫。這篇記錄想解決的問題、解法和架構選擇，以及部署到 Cloudflare Workers 的設定步驟。"
tags = ["Claude", "Cloudflare Workers", "Slack", "Obsidian", "AI"]
categories = ["筆記"]
series = ["客服小幫手"]
+++

## 想解決什麼問題

身為工程師，每天都會被客服問到一樣的問題：「這個錯誤是客人自己關掉的嗎？」「那個設定在哪裡開？」

答案其實都回答過好幾次了，但散落在 Slack 的歷史訊息裡。客服找不到，只好再問一次；我每次都要停下手邊的工作，重新解釋一遍。

我想要的是：

- **客服的問題，先交給 AI 回答**，我只處理真的沒遇過的問題
- **新問題回答過一次，就不用再回答第二次**
- 不要增加客服的負擔，他們**照常在 Slack 問就好**

## 怎麼解決

核心想法很簡單：把回答過的問題整理成知識庫，讓 AI 照著知識庫回答。重點是最後一步：**答不出來的問題，工程師回答後要自動存回知識庫**，這樣知識庫會在日常使用中自己長大，不需要一次寫完。

{{< mermaid >}}
flowchart TD
    A[客服在 Slack 發問] --> B[AI 讀知識庫回答]
    B -->|有把握| C[客服按「解決了」]
    B -->|沒把握| D[自動 @ 負責的工程師]
    D --> E[工程師回覆後按「存成 QA」]
    E --> F[AI 整理成筆記，開 PR 給負責人審核]
    F --> G[merge 後自動部署<br/>下次 AI 就會回答]
{{< /mermaid >}}

幾個設計上的原則：

- **只能根據知識庫回答**：寧可說「不知道，幫你找工程師」，也不要自己編答案
- **回答是給客服看的**：先給客服一句結論，再附一段可以直接複製給客人的回覆
- **個資自動遮掉**：客服一定會順手貼客人的 Email，靠提醒沒用，送給 AI 之前由程式先遮
- **大家一起維護**：每個工程師負責自己領域的 QA，用熟悉的 PR 流程審核

## 選什麼架構

| 角色 | 選擇 | 為什麼 |
|---|---|---|
| 提問入口 | **Slack** | 公司本來就用，客服不用換工具 |
| 知識庫 | **Obsidian 筆記放在 Git** | 就是 Markdown，AI 好讀；新增 QA 就是發 PR，可以多人審核 |
| 回答 | **Claude API** | 中文理解好，能照固定格式輸出，也會判斷自己有沒有把握 |
| 執行環境 | **Cloudflare Workers + Queues** | 不用顧主機；merge PR 就自動部署 |
| 部署 | **GitHub Actions** | 所有 key 集中放在 GitHub Secrets |

有兩個選擇值得多說一點：

**沒有用向量資料庫。** 一提到「AI + 知識庫」，很多人會想到向量資料庫。但我的知識庫一開始只有幾則 QA，直接把全部筆記交給 AI 最準、也最簡單。等知識庫大到放不下，再換成檢索也不遲。

**從自己架主機改成 Cloudflare Workers。** 一開始寫的版本要找一台主機常駐跑，後來改成 Workers：不用顧主機，也不用在自己電腦上跑任何東西。代價是 Slack 要求 3 秒內回應，但 AI 回答通常更久，所以中間加了一個 Queue：Worker 收到訊息先回 Slack「收到」，再把問題丟進 Queue 背景處理。

## 部署設定流水帳

程式寫完之後，真正花時間的是在 **Cloudflare、Slack、GitHub** 三個網站之間來回設定。順序很重要：Slack 需要知道 Worker 的網址，Worker 又需要 Slack 的 token，所以我是照下面這個順序做的。

最後要湊齊的是這 6 個 key，全部放進 GitHub Secrets：

| 名稱 | 從哪裡拿 |
|---|---|
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 帳號編號 |
| `CLOUDFLARE_API_TOKEN` | Cloudflare 部署用的 token |
| `ANTHROPIC_API_KEY` | Claude API key |
| `KNOWLEDGE_PR_TOKEN` | GitHub token（「存成 QA」自動開 PR 用） |
| `SLACK_SIGNING_SECRET` | Slack 用來證明「請求真的是 Slack 送的」 |
| `SLACK_BOT_TOKEN` | bot 發訊息用的 token |

### 1. Cloudflare：記下 Account ID 和子網域

到 Cloudflare 後台的 **Workers & Pages**，右側的帳號資訊有兩個值：

- **Account ID**：一串 32 個字的英數字。不用新建，註冊時就有了。也可以直接看網址列，`dash.cloudflare.com/` 後面那一段就是
- **Subdomain**：`xxx.workers.dev` 裡的 `xxx`，之後 bot 的網址會是 `https://support-bot.xxx.workers.dev`

這頁有一個 **Create application** 按鈕，**不要按**。Worker 會在第一次部署時自動建立。

### 2. Cloudflare：建立部署用的 API token

右上角頭像 → **Profile → API Tokens → Create Token** → 選 **Edit Cloudflare Workers** 範本。

- **Account Resources**：選自己的帳號
- **Zone Resources**：選一個自己的網域就好。bot 用的是免費的 `workers.dev` 網址，用不到網域，但這欄不能空著。只選一個，萬一 token 外流，影響範圍比較小
- **Permissions**：⚠️ **範本預設沒有 Queues 的權限**，要按 **Add more** 自己加一行 `Account → Queues → Edit`，不然部署會失敗
- **Client IP Address Filtering**：不要設，GitHub Actions 每次的 IP 都不一樣

按 **Create Token** 後，token **只會顯示這一次**，複製下來。

### 3. Anthropic：建立 Claude API key

到 Claude Console → **API Keys → Create Key**，取一個好認的名字，例如 `support-bot`。建議順便到 **Limits** 設一個每月花費上限，萬一 key 外流，損失也有上限。

### 4. GitHub：建立開 PR 用的 token

GitHub 頭像 → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**。

- **Repository access**：只選這個專案的 repo
- **Permissions**：**Contents** 和 **Pull requests** 都設成 **Read and write**

權限開越少越好，這把 token 只需要能在這個 repo 開 PR。

### 5. Slack：用 manifest 建立 App

到 Slack API 網站的 **Your Apps → Create New App → From a manifest**，選 workspace，切到 **YAML** 分頁，貼上事先寫好的設定檔。設定檔裡有 bot 的權限、要訂閱的事件，以及 Slack 要把事件送到哪個網址（就是第 1 步的子網域）。

這一步遇到幾個小狀況：

- **bot 的顯示名稱（`display_name`）不能用中文**，第一次貼上就報錯。App 名稱可以用中文，只有 `display_name` 要改成英文
- 網址驗證會失敗，因為 Worker 還沒部署。這是正常的，先繼續
- 建好後會跳出 Slack CLI 的教學（裝 CLI、`slack create`、`slack run`），**全部不用做**，直接按 **Go to App Settings**
- **Your Apps** 頁面上方的 Marketplace「I Agree」也不用按，那是上架到 Slack 公開市集才需要的

接著拿兩個值：

- **Signing Secret**：**Basic Information → App Credentials** → 按 **Show** 後複製。同一區還有 Client Secret、Verification Token，都不是要的
- **Bot User OAuth Token**：**OAuth & Permissions** 頁面最上面，`xoxb-` 開頭

同一頁下面的 **App-Level Tokens**（`xapp-` 開頭）是 Socket Mode 用的，部署在 Workers 用不到。旁邊的 **Token rotation** 也不要打開，那是開了就關不掉的設定。

### 6. GitHub：把 6 個 key 放進 Secrets

到 repo 的 **Settings → Secrets and variables → Actions**，停在 **Secrets** 分頁，按 **New repository secret**，一個一個新增。

要用 **Repository secrets**，不是上面那區的 Environment secrets，放錯地方部署時會讀不到。存進去之後 GitHub 不會再顯示內容，這是正常的。

### 7. 部署

merge 到 `main` 之後，GitHub Actions 會自動部署。到 **Actions** 分頁看 **Deploy** 有沒有出現綠色勾勾，大約 1 分鐘。第一次部署時，Queue 也會一起建立。

### 8. Slack：完成網址驗證

回到 Slack App 的 **Event Subscriptions**：

1. **Request URL** 旁邊按 **Retry**，出現綠色的 **Verified ✓**
2. 確認 **Subscribe to bot events** 裡有要的事件
3. 按 **Save Changes**；如果上方出現黃色提示要你 **reinstall**，就重新安裝一次

**Delayed Events** 不用開。它會在 bot 故障時把錯過的訊息慢慢補送，客服可能在好幾個小時後才收到很久以前問題的回答。

### 9. 建立測試頻道、邀請 bot

在 Slack 建一個**私人頻道**，只有自己在裡面，先拿來測試。

- 點頻道名稱，視窗最下面有 **Channel ID**（`C` 開頭）。用旁邊的按鈕複製，不要手打，`0` 和 `O` 很容易混淆。把它填進 Worker 的設定，這個頻道裡的每則訊息就會自動回答，不用 @
- 在頻道輸入 `/invite @support-bot`，把 bot 加進來

### 10. 之後加權限

後來要讓 bot 讀截圖，需要多一個 `files:read` 權限：**OAuth & Permissions → Bot Token Scopes → Add an OAuth Scope**。加完 Slack 會要求重新安裝 App，按 **Allow** 就好。Bot token 通常不會變，但還是比對一下，變了就要更新 GitHub Secrets。

## 上線後踩到的坑

設定都完成後，在 Slack 發問，bot 卻沒有反應。

到 Cloudflare 看紀錄時要注意，**要看 Worker 的 Logs**（Workers & Pages → support-bot → Logs），不是 Queue 的。Worker 的紀錄顯示 Slack 確實有把訊息送過來，但 bot 什麼都沒做，也沒留下原因。只好先加一行診斷紀錄，記下每則訊息「為什麼不回答」。結果找到兩個問題：

- 在指定頻道 @bot 時，我把這種事件略過了，以為另一種事件會處理
- 附截圖的訊息被 Slack 標成特殊類型（`file_share`），被當成「編輯訊息」一起略過

修好之後終於有回答，但很難讀：AI 回了一大段，還列出「如果是其他情況…」，長到被 Slack 折疊。Slack 的粗體符號遇到中文也不生效，直接把星號顯示出來。後來改成讓 AI 分欄位填內容（結論、回覆客人、內部說明），排版交給程式。

回頭看，最有用的是那行診斷紀錄。**讓系統說出「為什麼不做」，跟記錄「做了什麼」一樣重要。**

## 下一步

- 把更多舊的 QA 搬進知識庫，觀察哪些問題最常轉人工
- 統計每天問幾題、AI 自己答掉幾題
- 讓行銷的需求也走同一套知識庫
