# stagegate

[English](README.md) | 繁體中文

給 [Claude Code](https://claude.com/claude-code) 用的分階段、有 gate 把關的任務 pipeline：把有風險、具破壞性或需長時間執行的任務，驅動過一連串明確的階段（前提存活檢查 → 分流 → 需求確認 → 環境確認 → 工作規劃 → 執行工作 → 審查 → 驗證成果 → 觀察期 → 檢視改進）。階段之間以凍結的交接文件傳遞，關鍵處設 human gate，再加上跨圈次的 backlog，讓一次線性的執行接成能轉的循環。

以 Claude Code **skill** 的形式發佈：orchestrator 的操作指示寫在 [`SKILL.md`](SKILL.md)，參考文件按需載入（progressive disclosure）。

## 為什麼需要它

Agent 很會做事，卻不太會判斷什麼時候該停下來問。放著讓它獨自處理有風險的任務，它會自行腦補缺漏的需求、跳過備份，看到測試全綠就宣告成功。stagegate 用結構而非在 prompt 裡苦苦哀求來處理這件事：

1. **互動收束在同一層。** 只有 orchestrator（前景 session）跟使用者對話。自主執行的 worker 遇到歧義不猜測，而是回傳 `status: needs_input` 並列出明確的 `open_questions`，把控制權彈回人機介面層。
2. **階段之間靠凍結文件交接，不靠對話記憶。** 每個階段只讀取上游已 `approved` 的文件，並產出一份自己的文件。context 耗盡也不會遺失狀態，新的 session 可以從這些文件接續。
3. **Gate 有型別之分。** Human gate（需求確認、檢視改進，以及任務具破壞性時的工作規劃）、route gate（分流）、time gate（觀察期——「現在通過」不等於「下週還通過」），以及有機械式出場條件的 auto gate。
4. **破壞性工作在執行前必須先有退路。** 低可逆的任務會強制工作規劃產出 `rollback_plan` 與一個 checkpoint task；在還原點被驗證「確實能還原」之前，破壞性步驟一律被擋下——光是「備份檔存在」不算數。
5. **能力要實證，不能推定。** 進入工作規劃前，需求依賴的每一個外部 API、權限與配額，都要用一次最小的真實呼叫實際打過。文件說某功能存在，不代表你的帳號做得到；等執行完才發現，代價是整圈回滾。
6. **backlog 層讓它成為循環。** 每一圈檢視改進產出的改進項與升格事實，會 feed-forward 到下一圈的分流與環境確認階段。
7. **花成本之前先確認前提還在。** 當意圖指向某個既有目標時，必須先用一條真實的唯讀指令確認它仍然存在——沒被退役、沒被取代——才開始分流。這一步比 capability probe 更早，因為若有接 spec-driven development（SDD）的 proposal 步驟，它通常在需求 sign-off 後就觸發，早於 probe；少了這道前置檢查，可能整套 proposal／design／tasks 都燒在一個早已不存在的目標上。
8. **驗收條件凍結前必須可被證偽。** 當某條驗收條件帶有統計門檻時，需求 sign-off 前要由一個全新的 strong 模型 reviewer 回答兩個問題：它在什麼情況下會失敗？在預期的樣本量下門檻有沒有可能過？沒有失敗情境的條件是恆真的；n=25 時不可能通過的 gate 不算 gate。兩者在階段 1 偵測的成本很低，到檢視改進才發現就很貴。
9. **計畫要建立的東西，要先證明建得出來。** 會改動部署、基礎設施、憑證或 CI 設定的圈次，要跑一次規劃 preflight：每個釘死的 image tag、preset 或版本都要實際查證；每個新建物件都列成獨立的 task；由計畫從未接觸的伺服器負責驗證的物件（資料庫 role、schema），要做伺服器端接受度 probe——把 DDL 包在 `BEGIN … ROLLBACK` 裡跑，或用伺服器端 dry-run。plan 全綠只證明工具接受你的設定，不代表資料庫接受你挑的 role 名稱。會被 agent 權限層擋下的步驟（包括驗證時才需要的 Job 觸發）也在這時先標給使用者，不等 worker 做到一半才撞牆。
10. **新測試要先失敗過才算數。** 當某圈新增或改寫測試時，驗證階段會故意把受測程式改壞——反轉條件、`rc != 0` 判斷、差一錯誤——每支新測試都必須轉紅。對著改壞的程式仍然全綠的測試是恆真的，不管它在 CI 上添了多少綠勾勾。每支測試手工改壞一處就夠；這不是 mutation testing 框架，而是證明測試真的接在它宣稱要檢查的東西上、成本最低的方法。

## Pipeline 流程

```text
-1   premise check  orchestrator，條件式           ── 前置檢查：只在意圖指向既有目標時執行
0    triage         orchestrator，互動式           ── Route gate：fast | full（預設 full）
1    requirements   orchestrator，互動式           ── Human gate：使用者 sign-off；門檻屬統計性質時加跑 ac_review
2    environment    worker，唯讀偵察               ── capability probe 全數通過才能進入規劃
3    planning       worker                         ── 破壞性／低可逆時為 Human gate；基礎設施變更要跑 preflight
4    execution      worker                         ── 還原點落地前，破壞性步驟一律擋下
4.5  review         worker，僅限 code 圈次         ── 非 code 產物跳過
5    verification   worker，獨立於執行者           ── 新測試必須對故意改壞的版本失敗
5.5  observation    worker，排程                   ── Time gate：observe_until 之前不下判定
6    retrospective  orchestrator，互動式           ── Human gate：next_iteration
```

階段名保留英文，與 [`example/`](example/) 裡的交接文件檔名（`00-intake.yaml`、`01-requirements.yaml`……）對應。

- **前提前置檢查**：在決定 route 之前執行，而且只在意圖指名既有目標時才觸發。判定 `ABORT` 就當場結束這一圈——不產生任何下游文件、不觸發 SDD proposal 步驟——因為 proposal 步驟（若有接上）觸發得比階段 2 的 capability probe 早，無法指望它攔下已退役或已被取代的目標。
- **Fast-path**：分流可以讓小任務略過部分階段——但 route 只能升級、不能降級；破壞性／低可逆的任務，以及依賴從未實際打過的外部能力的任務，永遠不能走 fast-path。
- **緊迴圈**（同一圈內）：`needs_input`、審查要求重做，以及驗證／觀察失敗，會彈回適當的上游階段。
- **寬迴圈**（跨圈次）：檢視改進以 `next_iteration: continue | done | abort | spawn` 收尾；改進項此時已在 backlog 裡，等著下一圈取用。

## Repo 結構

| 檔案 | 內容 | 什麼時候讀 |
|------|------|------------|
| [`SKILL.md`](SKILL.md) | Orchestrator 操作手冊（skill 進入點） | 一律最先載入 |
| [`stages-and-gates.md`](stages-and-gates.md) | 9 階段 state machine（外加條件式的階段 -1 前提存活檢查）：owner、gate、流程圖、迴圈規則 | 想知道**流程怎麼跑** |
| [`handoff-schema.md`](handoff-schema.md) | 階段交接文件的 YAML schema 與不變量 | 想知道**文件怎麼寫** |
| [`backlog.md`](backlog.md) | 跨圈次持久層的 schema | 想知道**圈次之間怎麼銜接** |
| [`worker-mapping.md`](worker-mapping.md) | 階段與 worker agent 的對應、模型選擇、委派協定 | 想知道**誰來做事** |
| [`example/`](example/) | 完整的實作範例：一個小任務的全部 9 份交接文件 | 想看**實際長什麼樣子** |

建議閱讀順序：先看 `stages-and-gates.md` 的角色與階段總覽兩節 → `example/README.md` → 再回到 `handoff-schema.md` 與 `backlog.md` 看欄位細節。

所有參考文件與範例皆為英文；本檔僅翻譯 landing page。

## 安裝

clone 到 Claude Code 的 skills 目錄：

```bash
git clone https://github.com/NeoWYM/stagegate.git ~/.claude/skills/stagegate
```

若只給單一專案用，則 clone 到 `<repo>/.claude/skills/stagegate`。

之後在 Claude Code 裡用自然語言觸發即可（「用 pipeline 跑這個」／"run this through the pipeline"、「照分階段流程做」／"use the staged process for this"），或直接開始討論一項有風險的變更——skill 的 description 已涵蓋觸發條件。

## 依你的環境調整

這個 skill 刻意做成與環境無關。有三個地方你會想客製：

- **Worker**（`worker-mapping.md`）：如果你有自己的唯讀 subagent（基礎設施健康檢查、DB 檢查器……），把偵察與驗證階段對應過去。內建的 `Explore`／`Plan`／`general-purpose` agent 開箱即可使用。
- **審查的權限邊界**：審查階段要交給*工具權限*本身就是唯讀的 agent。在 prompt 裡寫「請不要修改任何東西」不構成邊界；工具 allowlist 才是。
- **Pipeline 目錄**：替每圈的 `NN-<stage>.yaml` 文件與 `backlog.md` 選一個固定位置（例如工作目錄下的 `pipelines/<task-slug>/`）。

## 設計筆記

- 互動式階段（0、1、6）永遠不委派：在 Claude Code 裡，subagent 無法跟使用者來回對話，只能回傳一則最終訊息。凡是需要對話的，都必須留在前景 session。
- 刻意**不設通用的 executor worker**。執行工作高度依賴領域；一體適用的 executor 最後會變成一個什麼都做、什麼都做不好的巨型 prompt。orchestrator 要嘛自己執行，要嘛逐 task 委派給該領域的 agent。
- 驗證與審查分成兩個階段，是因為它們回答的問題不同：驗證是黑箱（「它能動嗎？」），審查是白箱（「它寫得對嗎？是需求真正要的做法嗎？」）。測試全綠時，照樣可能藏著 hardcode 的值、漏掉的邊界情況與安全問題。

## 授權

[MIT](LICENSE)
