# 用 Claude Code Skills 自動化股票篩選

![](./assets/stock_filter_automation.png)

原文: [Claude Code Skills で株スクリーニングを自動化した話 Vol.1【Python × yfinance × バイブコーディング】](https://qiita.com/okikusan-public/items/61100a5b1aa8d752ae24)

## 為什麼需要自動化投資分析

我使用 Claude Code Skills 與 yfinance，打造了一套從股票篩選到投資組合管理的投資分析自動化系統。以 Python × Vibecoding 的方式，用 5 個 Skill 覆蓋個人投資者的完整投資工作流程。

身為個人投資者，操作日股與美股時，「尋找低估標的」這件事說來枯燥卻相當麻煩。在券商的篩選器過濾後，還要逐一確認篩出的股票，再考量整體投資組合的平衡——這種反覆循環讓我很想找到改善方法。

向 ChatGPT 或 Claude 的網頁版要求「幫我找價值被低估的股票」，它們確實能做到一定程度。但一旦想整合進投資決策的工作流程，就遭遇了 4 道高牆：

- **回應與資料來源不穩定**: 網頁搜尋每檔股票耗時數十秒，參考文章每次都不同，無法用於定點觀測
- **脈絡消失**: 上週的篩選結果與買賣理由，下一個對話階段就全部遺忘
- **缺乏重現性**: 每個階段 PER 閾值與評分微妙不同，無法在沒有重現性的機制上累積投資判斷
- **處理量的瓶頸**: 委託做 10 檔健康檢查，輸出卻在 3～4 檔就中斷，但「不遺漏地檢查全部標的」才是有意義的

Claude Code 的 **Skills** 解決了以上全部 4 個問題：

- **回應與資料來源**: 將邏輯固定在 Python 腳本中，透過 yfinance API 統一取得資料，每次都以相同方式從相同來源取值
- **脈絡消失**: 投資組合存 CSV、閾值存 YAML、設計決策存 CLAUDE.md，跨對話階段仍能延續脈絡
- **重現性**: 篩選邏輯作為腳本固定下來，相同輸入就執行相同處理
- **處理量**: 以 Python 程序而非 LLM 輸出 token 執行，無上限限制。東京證交所全上市股（Prime・Standard・Growth 合計約 1,000～2,000 檔）一次批次篩選，數十秒內完成

最初只打算做篩選功能，但「也想要報告」「也想管理投資組合」「也需要健康檢查」「還需要壓力測試」……需求不斷擴大，不知不覺就發展成 5 個 Skill 組成的系統。

本文主要介紹這套系統的設計思路、技術選型判斷。

## 什麼是 Claude Code Skills

Claude Code 具備一種可以自製斜線指令(slash command)的機制，稱為 **Skills**。在 Skill 定義檔（`SKILL.md`）中寫好 Skill 的名稱、說明、引數規則、允許的工具，Claude Code 就能解讀自然語言輸入並執行腳本。

[![](https://storage.googleapis.com/zenn-user-upload/883c67828cf4-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F883c67828cf4-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=da3319d7d9c0d3f40ec05a062101341c)

重點在於，**使用者只需以自然語言對話即可**，連斜線指令都不用輸入。Claude Code 會理解使用者的意圖，自動選擇適當的 Skill 並執行。

例如直接說「幫我找日股高配息的」，Claude Code 就會從脈絡判斷需要來選擇可以完成使用者目標的 Skill 與適當設定地區與預設條件後執行腳本。問「投資組合狀況怎麼樣？」就會啟動健康檢查；說「給我看豐田的報告」就會生成個股報告。

也就是說，Skills 並非「命令列工具」，而是 **Claude Code 依情況靈活運用的工具箱**。使用者不需要記住任何指令體系，只需在投資的脈絡下自然對話，背後就會自動執行適當的腳本。這種 {==「自然語言 → Skill 選擇 → 腳本執行 → 結果解讀」==} 的完整流程，正是 Vibecoding 體驗的本質，也是與傳統 CLI 工具最決定性的差異。

另一個重要特性是 **重現性**。邏輯固定在 Python 腳本中，保證今天與上個月的篩選是以相同標準執行的。

## 自動化投資分析的 5 個 Skills

這次打造的 Skill 共有 5 個：

| # | Skill | 功能 |
| --- | --- | --- |
| 1 | `screen-stocks` (篩選) | 搜尋低估股（4 個引擎、7 個預設、支援 yfinance EquityQuery 的 60+ 個交易所） |
| 2 | `stock-report` (個股報告) | 指定股票代號生成財務分析報告 |
| 3 | `stock-portfolio` (投資組合管理) | 買賣記錄, 損益顯示, 結構分析, 健康檢查, 預估殖利率, 再平衡建議 |
| 4 | `stress-test` (壓力測試) | 以 8 個情境驗證投資組合弱點（集中度, 相關性, VaR）|
| 5 | `watchlist` (自選清單) | 管理感興趣標的的清單 |

涵蓋了「篩選找到標的 → 報告確認細節 → 加入自選清單 → 投資組合記錄買賣 → 健康檢查持續監控 → 壓力測試掌握風險」的完整投資工作流程。

[![](https://storage.googleapis.com/zenn-user-upload/a8183355d6ce-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2Fa8183355d6ce-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=d82b9e79546a33ac7520103c7e848cbf)


## 系統設計：三層架構與資料持久化

### 為何分成三層

系統分離成 **Skills 層（介面）→ Core 層（商業邏輯）→ Data 層（資料取得）** 三層。

- **Skills 層** 僅負責介面，是只解析引數、呼叫腳本的薄層
- **Core 層** 集中商業邏輯，配置了篩選引擎、評分、健康檢查、情境分析等 17 個模組，在 5 個 Skill 之間共用邏輯。例如「變化分數」的計算邏輯，會被篩選（Alpha 預設）和健康檢查雙方呼叫
- **Data 層** 集中資料取得，不直接呼叫 yfinance，一律經由包裝層，統一處理快取（24 小時 TTL）、速率限制（API 間 1 秒延遲）、異常值清洗（排除配息率 > 15% 或 PBR < 0.1 等）

這種分離設計讓「新增 Skill」時可重用 Core 邏輯，「更換資料來源」時只需替換 Data 層。

### 資料持久化策略

針對「**脈絡消失**」的問題，以適當格式持久化三類資料：

| 資料 | 格式 | 理由 |
| --- | --- | --- |
| 投資組合（持有標的・買賣歷史） | CSV | 人工直接編輯・確認方便 |
| 篩選閾值, 交易所定義 | YAML | 以高可讀性管理結構化設定 |
| 設計決策, 架構 | CLAUDE.md | 讓 Claude Code 在新對話階段也能理解專案脈絡 |
| 標的資料快取 | JSON | 直接儲存 API 回應（24 小時 TTL） |

!!! tip
    挑戰:
      - **脈絡消失**: 上週的篩選結果與買賣理由，下一個對話階段就全部遺忘
    解決:
      - **脈絡消失**: 投資組合存 CSV、閾值存 YAML、設計決策存 CLAUDE.md，跨對話階段仍能延續脈絡

其中 `CLAUDE.md` 尤為重要，詳細記錄了篩選引擎的運作規格、ETF 判斷規則、資料取得模式等。如此一來，即使在新的對話階段開啟 Claude Code，也能在理解專案脈絡的狀態下繼續開發。

## 技術選型與邏輯

### 資料來源：yfinance

這套系統的基礎是 **[yfinance](https://github.com/ranaroussi/yfinance)**（Yahoo Finance 的 Python 函式庫）。免費且無需 API 金鑰，可取得個人投資者所需的絕大部分資料。

| 資料種類 | 可取得內容 | 用途 |
| --- | --- | --- |
| 股價與估值 | PER（本益比）, PBR（股價淨值比）, 配息率, 市值, 52 週高低價 | 篩選、報告 |
| 財務報表 | 損益表, 資產負債表, 現金流量表 | 變化分數（Alpha）、健康檢查 |
| 分析師共識 | 目標股價（高/均/低）, 評等, 分析師人數 | 預估殖利率（3 情境） |
| 價格歷史 | 日/週/月 OHLCV（最多數十年） | 技術分析、VaR、相關性分析 |
| 新聞 | 官方媒體新聞文章（標題・URL・摘要） | 預估殖利率的定性資訊 |
| EquityQuery | 以條件批次搜尋標的（yfinance 端支援 60+ 個交易所） | 篩選（全市場一次） |

特別強大的是 **EquityQuery**，輸入「PER < 15 且配息率 > 3% 且東證上市」這類條件，就能批次返回符合的標的。不需事先準備股票清單，可以不漏網地篩選東證 Prime・Standard・Growth 全部標的。這是與網頁版逐一查詢體驗最大的差距。

yfinance 雖然免費，但資料存在特殊習性（ETF 銷售歷史以空清單返回、配息率有時以百分比數值返回等），因此設計在 Data 層統一進行異常值清洗。

### 篩選：4 個引擎

依目的備有 4 個篩選引擎：

| 引擎 | 方式 | 用途 |
| --- | --- | --- |
| QueryScreener | yfinance EquityQuery API 批次取得 → 以 Value 分數排名 | 基本低估股搜尋（value, high-dividend 等 5 個預設） |
| PullbackScreener | EquityQuery → 以 RSI/布林通道判斷回調 → Value 分數 | 想在上升趨勢中短期回調時進場 |
| AlphaScreener | EquityQuery（門檻過濾）→ 變化分數（3/4 指標）→ 回調判斷 → 雙軸分數 | 低估且有業績改善跡象的標的（4 段管線） |
| ValueScreener | 逐一取得標的清單 → 過濾 → 評分 | 舊式方法（為相容性保留） |

Value 分數配分為 **PER（25 分）+ PBR（25 分）+ 配息率（20 分）+ ROE（15 分）+ 營收成長率（15 分）= 滿分 100 分**，各指標以 0～滿分範圍評分後加總排名。

AlphaScreener 的「變化分數」，以應計項目（利益品質）・營收加速度・FCF 毛利率變化・ROE 趨勢等 4 個指標，量化「業績是否正往好的方向變化」。在確認低估的同時觀察變化方向，以此迴避「長期低估不動的陷阱（價值陷阱）」。

### 健康檢查：技術面 × 基本面雙軸判斷

判斷持有標的「投資假設是否仍然有效」的健康檢查，是三級警報系統：

| 警報 | 技術面條件 | 基本面條件 | 行動 |
| --- | --- | --- | --- |
| 早期預警 | 跌破 SMA50 / RSI 急跌 | ── | 留意觀察 |
| 注意 | SMA50 接近 SMA200 | 變化分數 1 項指標惡化 | 考慮部分獲利了結 |
| 撤退 | 死亡交叉 | 變化分數多項惡化 | 考慮撤出 |

設計上的重要決策是，**撤退訊號必須同時要求技術面崩潰與基本面惡化**。即使出現死亡交叉，若基本面良好，仍只停在「注意」等級。僅憑一時的供需崩潰就發出賣出訊號會造成過度反應，因此設定必須兩面都有依據的條件。

ETF（黃金 ETF、債券 ETF 等）自動判定為基本面分析對象之外，僅以技術面評估。

[![](https://storage.googleapis.com/zenn-user-upload/6ecfa9e4403e-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F6ecfa9e4403e-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=0452075dcc0cef062447566a25001c93)

### 預估殖利率：分析師共識 + 歷史報酬

以 `樂觀` / `基本` / `悲觀` 三個情境，呈現 12 個月後的預期報酬。個股與 ETF 採用不同方式處理。

**個股** 使用 yfinance 可取得的分析師目標股價（高 / 均 / 低）。分析師人數較少時（未滿 3 名），目標價的價差往往偏窄，因此自動擴大價差以區分 3 個情境。

**ETF** 因無分析師追蹤，從過去 2 年的月度報酬計算 CAGR（複利年成長率），以標準差分歧情境。設定上限以防異常值造成過高估計。

此外，可選擇性加入 yfinance 官方新聞，以及 Grok API（X Search）的 SNS 情感分析。Grok API 設計為環境變數未設定時自動跳過（Graceful degradation）。

[![](https://storage.googleapis.com/zenn-user-upload/7bff7e428aed-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F7bff7e428aed-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=75cb51df1b42a48e27340932167130cb)

### 壓力測試：情境式風險分析

以 8 個預先定義的情境（三殺、科技崩跌、日圓升值等）驗證投資組合弱點。

分析以管線結構化進行：集中度分析（HHI）→ 衝擊敏感度 → 各情境衝擊估計 → 相關性分析（皮爾森相關 + 總體因子分解）→ VaR（95%/99%）→ 因果連鎖分析 → 建議行動。

技術重點：ETF 自動分類為資產類別（黃金・長期債・股票收益等），並為各情境套用適當的衝擊係數。例如「三殺」情境中，黃金 ETF 作為避險資產，計算出與股票相反方向的衝擊。

建議行動由規則引擎自動生成，再由 Claude 加入考量投資組合脈絡的補充說明。定量分析（計算）與定性分析（LLM 解讀）分離，充分發揮各自優勢的設計。

[![](https://storage.googleapis.com/zenn-user-upload/80598dba5981-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F80598dba5981-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=6f6d514472a6f4c40418c4bc9f8e636d)

個股報告是指定股票代號生成財務摘要的功能，自選清單是以 JSON 永久管理感興趣標的的功能。雖然都是簡單的輔助 Skill，但扮演著串連「篩選 → 報告確認 → 加入自選清單」流程的重要角色。

[![](https://storage.googleapis.com/zenn-user-upload/58a0bd7bd3d3-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F58a0bd7bd3d3-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=a7f10d5f0651cae371f337a9f6ec05d2)

投資組合快照功能可一覽持有標的的當前評估值與損益。

[![](https://storage.googleapis.com/zenn-user-upload/3a62e8e58806-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F3a62e8e58806-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=43b6e420f12fa31df382b3ee11f5cd0a)

## Agents / Teams：並行搜尋、並行驗證

Claude Code 具備以團隊形式並行啟動多個代理程式的 **Teams** 功能，與篩選系統的配合堪稱絕配。

### 篩選的並行化

「想同時在日本・美國・東南亞・香港 4 個地區尋找低估股」時，循序執行每個地區需 30 秒～1 分鐘，合計要等好幾分鐘。使用 Teams，4 個代理程式 **分別同時探索不同地區**，彙整結果後返回。

與 1 個代理程式依序處理相比，**實際等待時間接近 1/N**。正因為這個系統支援 yfinance 的 60 個以上交易所，並行化的效益才能如此顯著。透過 yfinance 還可取得分析師共識資訊（目標股價・評等・分析師人數），讓「同時搜尋全球低估股，依與共識目標的乖離幅度排列」這種全面搜尋成為現實。

### 綜合測試的並行化

同樣機制也用於品質管理。並行啟動 4 個測試代理程式，一次執行全部 Skill 的整合測試。

- 篩選測試員 → 4 個模式（日本 Value、美國高配息、日本 Alpha、美國回調）
- 報告測試員 → 報告 4 個模式 + 自選清單 CRUD 6 個模式
- 投資組合測試員 → 5 個子指令（list, snapshot, analyze, health, forecast）
- 壓力測試員 → 4 個模式（混合、指定情境、僅 ETF、指定權重）

結果 **23 個測試全部通過**，過程中發現 2 個輕微 bug，並自動建立 Linear 工單。

[![](https://storage.googleapis.com/zenn-user-upload/70525a34e58c-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F70525a34e58c-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=01022cbf6843f77f548db4a38802432a)

## 開發的真實情況

### 遇到的陷阱

介紹在製作用 Skills 運作的東西時，實際碰到的困難點，都是起因於 yfinance 資料特性的問題。

**ETF 判斷的陷阱。** 在將 ETF 排除於健康檢查之外的判斷邏輯中，沒注意到 yfinance 對 ETF 的銷售歷史返回「空清單」，導致 ETF 被誤判為個股。改用 `bool()` 的 truthiness 檢查取代 `is not None` 後解決。外部 API 返回值如何表達「無資料」，往往是光看文件無法理解的教訓。

**ETF 報酬的年化換算。** 以月度報酬中位數單純乘以 12 的單利計算，導致黃金 ETF 的預估報酬出現 +86% 這種明顯高估。修正為 CAGR（複利年率），並將取得期間從 6 個月擴大至 2 年，從而得到穩定的估計值。

**分析師人數過少時情境崩潰。** 分析師在 2 名以下的標的，目標股價的高 / 均 / 低全部變成相同數值，使 3 個情境失去意義。改為人數過少時自動加入價差以產生分歧。

### Vibecoding 的開發流程

**iTerm 多視窗 × worktree 並行開發。** 實際開發時，iTerm2 視窗按角色分開同時並行作業：

- **PdM 視窗** ── 在 Linear 建立 Issue，審查完成的實作，有問題就返回回饋，扮演產品管理角色
- **開發視窗** ── 在 feature 分支推進實作，與 Claude Code 對話同時寫程式
- **測試視窗** ── 執行測試與動作確認

Claude Code 是終端機為基礎，因此「多視窗同時與 Claude 對話」這種風格自然地相互契合。在 PdM 視窗建立 Issue 整理需求，把實作委派給開發視窗的 Claude，完成後在 PdM 視窗審查並回饋——這正是 **Vibecoding**。自己只需以自然語言表達想做的事，AI 就會負責設計・實作・測試。一人運作的敏捷開發流程就此成立。

進一步使用 **git worktree**，可以把同一個 repository 的不同分支 checkout 到不同目錄，因此可以在一個視窗推進 feature 分支開發，同時在另一個視窗跑 main 分支測試。每個 Issue 切一個 worktree，獨立推進開發・測試・合併，對個人開發的效率提升相當顯著。

**Linear 整合。** 透過 Claude Code 的 MCP（Model Context Protocol）連接 Linear，直接從 PdM 視窗建立・更新 Issue。「這是 bug」→ 建立 Issue → 開發視窗修正 → 測試 → PdM 視窗審查 → 更新為 Done，全部在終端機內完成。

[![](https://storage.googleapis.com/zenn-user-upload/6fb6b5f520a4-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F6fb6b5f520a4-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=714ba12a8da2a2f7833c1a6e78ffe705)

[![](https://storage.googleapis.com/zenn-user-upload/087daf340ddd-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2F087daf340ddd-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=fe81285e73b4f3365fc965f59089a42f)

**單元測試。** 使用 pytest 維持 **740 個以上**的測試。全部測試約 20 秒完成，每次修改都能輕鬆執行。由於對外部 API 進行了 mock，無需網路連線即可高速執行。這些單元測試的存在，是能安心進行重構的基礎。即使把程式碼改善委派給 Claude Code，只要測試通過，就能立即確認現有行為沒有被破壞。以 AI 主導開發的風格中，「委派沒問題嗎」的不安如影隨形，測試就是這道安全網。

[![](https://storage.googleapis.com/zenn-user-upload/cfbd1c82b42d-20260214.png)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fstorage.googleapis.com%2Fzenn-user-upload%2Fcfbd1c82b42d-20260214.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=947f53272afc72488725f70c93a9d7d9)

### Skills 開發獲得的知見

**1. `SKILL.md` 要寫成「介面規格書」。** 明確定義引數解讀規則與輸出格式，Claude 就能精確呼叫腳本。模糊的定義會導致模糊的執行結果。

**2. 邏輯要放在 Skill 之外。** Skills 的腳本專注於入口點，商業邏輯分離到其他模組。如此一來，Skill 之間可共用邏輯，測試也更容易撰寫。

**3. `CLAUDE.md` 要寫上架構。** 因為 Claude Code 在修改程式碼時會參考，詳細記載模組間的依賴關係與資料流，可以提升修改精確度。

**4. 如何與脈絡上限共存。** Claude Code 的對話階段拉長了，同樣會觸及脈絡上限。處理思路是「知識的層次化」：可邏輯化的部分寫進 Python 程式碼，能以規則整理的部分彙整在 Skills 的 `SKILL.md`，整個專案的設計決策寫在 `CLAUDE.md` 或 Rules。如此一來，即使在新的對話階段，Claude Code 也能讀 `CLAUDE.md` 理解脈絡後開始作業。脈絡只需載入「現在想做的事」，定期重構整理知識的循環也能持續運作。

**5. 意識到 Graceful degradation。** 外部 API（Grok API 等）設計為未設定時自動跳過，可防止環境差異造成的錯誤。「沒有它也能動，有的話更好」是理想。

## 總結：Claude Code Skills 讓投資分析自動化到這種程度

作為 Vibecoding 的實踐案例，使用 Claude Code Skills，以自然語言介面建構了「可互動使用的投資分析系統」。

- 以 Skills 的機制，結構性地解決了網頁版體感到的 4 個問題（回應・脈絡消失・重現性・處理量）
- 只需以自然語言對話，Claude 就會選擇並執行適當的 Skill，不需要記住任何指令體系
- 邏輯固定在腳本中，保證每次篩選都有相同品質
- 透過 Teams 並行執行，對 yfinance 支援的 60 個以上交易所同時篩選、收集共識資訊成為現實
- Linear 整合・740 個以上測試自動執行，開發・運維・品質管理也在 Claude Code 內一手包辦

{== Skills 是將「會寫程式的 AI」轉變為「具備可用工具的 AI」的機制。邏輯一旦打造好，之後只需以自然語言對話，就能得到具重現性的分析。==}

回顧來看，這個專案 **整個開發循環都在 AI 與終端機中自給自足**。在 Linear 整理需求，在 iTerm 多視窗與 Claude Code 對話同時實作，以 worktree 並行跑測試，發現 bug 就自動建立 Linear 工單修正，以 Teams 並行執行整合測試，合併後更新為 Done。幾乎沒有開瀏覽器的場合。AI 不只是寫程式，連需求管理・品質管理・專案管理都能一條龍承擔的世界，已經就在手邊。

不只限於個人投資者的工作流程改善，這是一種可以應用在所有反覆進行定型分析或處理業務上的方法論。

老實說，要將這個做成 App 讓更多人使用——我認為是困難的。Skills 是依賴 Claude Code 本地環境的機制，並非設計成能以網路服務形式提供。不過，原本的目的就是「提高自己的投資判斷效率」。作為自己專用的工具，手邊有了一套替自己賺錢的機制，這樣就已經是充分的成果了。

今後打算與 PdM 視窗的 Claude 持續討論，將所有金融相關知識系統性地體系化成 Skills。不只是股票，還有債券、匯率、總體經濟指標、稅制——投資決策所需的知識相當廣泛。將它們一個一個落實成 Skill，自己的金融素養也隨之提升的實感讓人振奮。

{== 打造 Skills 需要深度理解目標領域，因此不只是「委派給 AI」，也成為了「與 AI 一起學習」的工具。這種積累，會讓與 AI 共同工作的環境越來越豐富。==}

有興趣的朋友歡迎試試看，原始碼已在 GitHub 公開:

- [stock_skills](https://github.com/okikusan-public/stock_skills)
- [stock_skills 英文版](https://github.com/erhwenkuo/stock_skills)

> **注意：** 投資為個人責任。本系統的輸出並非投資建議。此外，雖然在 GitHub 公開了原始碼，但不接受任何關於投資成果的諮詢或支援，請僅作為 Skills 的技術參考。

**📘 系列文章**:

- [Vol.1：用 Claude Code Skills 自動化股票篩選](https://qiita.com/okikusan-public/items/61100a5b1aa8d752ae24)
- [Vol.2：用 Claude Code Skills 打造「越用越聰明」的投資分析 AI](https://qiita.com/okikusan-public/items/405949f83e8a39a49566)
- [Vol.3：Claude Code Skills × 投資分析 Vol.3 — 處方箋引擎・逆勢偵測・標的分群](https://qiita.com/okikusan-public/items/1765d6afb8c548f019f1)
