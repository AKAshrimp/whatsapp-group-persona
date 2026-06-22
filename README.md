# 朋友 Skill

把溫暖的群友，變成冷冰冰的 skill。  
靈感來自 前任 skill，但這次不翻舊情，只翻群聊語氣。

朋友 Skill 是一個 WhatsApp 群友 persona 框架，把群聊裡的短句、笑位、emoji、語氣和熟人感，蒸餾成 chatbot / RAG / agent 可用的安全 skill。

這不是 LoRA，也不是模型微調。它抽象的是一位群友的說話節奏、語氣強弱、語言混用、幽默方式和互動模式，同時避免複製舊訊息、翻舊事或冒充真人。

## 核心特點

| 能力 | 用途 |
| --- | --- |
| WhatsApp `.txt` 解析 | 支援常見 WhatsApp export 格式和多行訊息 |
| 三層 persona 生成 | 先本地統計，再場景分桶，最後生成正式 skill |
| 上下文窗口 | 分析目標成員如何回應前後群聊語境 |
| 人名排除 | 避免把群友英文名誤判成英文口頭習慣 |
| 安全規則 | 預設不輸出舊原句、不翻舊事、不冒充真人 |
| 可修正 persona | 支援使用者補充主觀印象和 correction |

## 三層工作流

```text
WhatsApp export txt
  ↓
Layer 1：本地統計
  解析訊息數、語言比例、句長、emoji、英文短詞
  ↓
Layer 2：場景分桶
  整理被 tag、追問、安排資訊、接梗、拒絕等互動模式
  ↓
Layer 3：正式 Skill
  產生 persona.md、memories.md、style_rules.md、meta.json、SKILL.md
```

這樣做可以避免一開始就把整份聊天丟給 LLM，降低 token 成本，也減少舊事件被寫進 persona 的風險。

## 專案結構

```text
prompts/
  intake.md                  # 收集成員、群組、背景、禁區
  memories_analyzer.md        # 分析群組背景
  persona_analyzer.md         # 分析目標群友風格
  memories_builder.md         # 整理群組背景區塊
  persona_builder.md          # 產生單一 member markdown
  merger.md                   # 追加新聊天記錄後合併
  correction_handler.md       # 人手修正 Persona

tools/
  whatsapp_txt_parser.py       # 解析 WhatsApp 匯出 txt
  context_persona_extractor.py # 第一層統計和上下文切片
  skill_writer.py              # 產生本地 Persona Skill 檔案
  version_manager.py           # 版本回滾工具

exes/
  example_xiaomei/             # 假資料範例，保留於 repo
```

生成的私人 persona 預設放在 `exes/{member_slug}/`，並被 `.gitignore` 忽略。

## 輸出結構

```text
exes/{member_slug}/
  SKILL.md
  persona.md
  memories.md
  style_rules.md
  meta.json
  analysis/
    {member_slug}-first-layer.json
    {member_slug}-second-layer-summary.md
```

| 檔案 | 說明 |
| --- | --- |
| `SKILL.md` | 可直接交給 agent / chatbot 讀的完整入口 |
| `persona.md` | 說話風格、互動方式、幽默和情緒反應 |
| `memories.md` | 群組背景和抽象互動記憶 |
| `style_rules.md` | 安全邊界和生成控制 |
| `meta.json` | 名稱、slug、來源、版本、風格標籤 |
| `analysis/` | 中間分析材料，方便回查；不建議 runtime 每次載入 |

## 為什麼 `SKILL.md` 和 `persona.md` 會相似

`SKILL.md` 是完整入口，通常會嵌入 `persona.md`、`memories.md` 和 `style_rules.md` 的核心內容。這樣即使 runtime 只載入 `SKILL.md`，也不會缺少必要規則。

`persona.md` 是方便人手編輯的源文件。若要調整語氣、人味、回覆示例，優先改 `persona.md`，再同步到 `SKILL.md`。如果你的系統可以同時讀多個檔案，也可以讓 `SKILL.md` 保持較短，只放索引和執行規則。

## 快速開始

### 1. 安裝依賴

```powershell
pip install -r requirements.txt
```

`pypinyin` 只用於中文名字轉 slug。核心 parser 不依賴外部 LLM。

### 2. 第一層：本地統計

```powershell
python tools/context_persona_extractor.py `
  --file "chat.txt" `
  --group "星河測試群" `
  --member "Mika" `
  --output "exes/mika/analysis/mika-first-layer.json" `
  --before 3 `
  --after 2 `
  --sample 40 `
  --exclude-word "alex" `
  --exclude-word "blake"
```

這一步會輸出：

- 目標成員訊息數
- 繁體中文 / 英文 / 混合語言比例
- 短句比例
- 問號、感嘆號、emoji 使用
- 常見英文短詞
- 少量上下文窗口樣本

### 3. 排除人名和噪音詞

`top_english_words` 只應保留語言習慣，例如 `ok`、`me`、`can`、`when`、`no`。如果混入群友英文名、URL 或組織名，請用 `--exclude-word` 排除。

常見應排除項：

- 群友英文名或代號，例如 `alex`、`blake`、`casey`、`devon`
- 目標成員自己的英文名片段
- URL 或網站噪音，例如 `http`、`https`、`www`、`com`

### 4. 第二層：場景分桶

第二層建議由本地統計和少量代表片段產生。常見分桶：

- 超短反應 / 確認
- 一般短句敘述
- 主動點名 / 拉人入話題
- 工作班次 / 安排資訊
- 否定 / 拒絕 / 劃界線
- 接梗 / 笑場

輸出建議放在：

```text
exes/{member_slug}/analysis/{member_slug}-second-layer-summary.md
```

這份檔案不是正式 skill，只是分析材料。

### 5. 第三層：正式 Skill

把第二層摘要、人手印象和安全規則合併成：

```text
exes/{member_slug}/SKILL.md
exes/{member_slug}/persona.md
exes/{member_slug}/memories.md
exes/{member_slug}/style_rules.md
exes/{member_slug}/meta.json
```

如果用 prompt pipeline，可以按以下順序：

```text
01 intake
02 memories_analyzer
03 persona_analyzer
04 memories_builder
05 persona_builder
06 merger             # 有新資料時才用
07 correction_handler # 使用者覺得不像時再用
```

## 人手補充人味

統計會抓到骨架，但不一定能抓到一個人的溫度。建議在 `persona.md` 補一層：

```markdown
## 使用者主觀印象

- 性格印象：
- 溫暖感來自：
- 她關心人時通常怎樣：
- 熟人之間的玩笑邊界：
- 哪些地方不能寫得太冷：
- 哪些地方不能寫得太誇張：
- 偏好或禁忌：
```

也可以寫入 `Correction 記錄`：

```markdown
## Correction 記錄

- [2026-06-22] 情境：整體人味
  - 錯誤表現：太像冷冰冰規則，不像溫暖群友
  - 修正規則：保留短句和冷面幽默，但加入自然關心、輕鬆接話和熟人感
  - 作用範圍：global
```

## 回覆例子和舊語境

正式 skill 可以放少量例子，但建議分成兩類：

1. **仿寫示例**
   - 展示某種場景下應該怎樣回。
   - 可以放在 `persona.md` 和 `SKILL.md`。
   - 例子應該是新寫的，不要直接複製整段舊聊天。

2. **少量真實短句**
   - 只用來校準節奏、長度、emoji 密度和英文混用方式。
   - 只選很短、沒有私密語境的句子。
   - 不應整理成大量 phrase bank，否則 chatbot 容易復讀舊訊息。

舊訊息和上下文可以作參考，但應抽象成規則：

```text
不好：遇到某場景就複製某句舊原句
好：遇到某場景時，用短句先確認，再補一句資料或輕量吐槽
```

如果需要 persona 回覆長一點，不要改成正式長文。應使用「短句連發」：

```text
可以
但要睇下佢幾點 confirm
太夜就算
```

## Chatbot token 使用建議

不要每次聊天都把完整 `SKILL.md`、`memories.md`、大量例子和歷史材料全部塞進 prompt。這會一直消耗大量 token。

建議拆成三種載入層級：

| 載入層級 | 使用時機 | 內容 |
| --- | --- | --- |
| Runtime profile | 每次都放 | 10 至 20 行精簡規則 |
| Examples | 需要更像時才放 | 少量仿寫示例和短句校準 |
| Full skill | 初始化或 debug | 完整 `SKILL.md` |

實際接入 chatbot 時，推薦每次只放 runtime profile，然後按需要從 examples 或 memories 取少量相關內容。

## 安全規則

- 預設不要輸出舊原句。
- 只有使用者明確要求原句、逐字、引用或 exact wording 時，才允許短引用歷史原句。
- 普通聊天只模仿風格，不複製歷史訊息。
- 不主動翻舊事。
- 不冒充真人。
- 不把私密資訊當玩笑素材。
- 不要把真實 WhatsApp export、真實成員名、真實群名或真實 persona push 到公開 repo。

## 測試

```powershell
python -m unittest discover -s tools -p "test_*.py" -v
```

目前測試覆蓋：

- WhatsApp txt parser
- 第一層上下文提取工具
- Persona writer
- 7 個 prompt 是否保留必要安全規則
- 假名 / 匿名化範例

## Git 安全

`.gitignore` 預設忽略：

- `exes/*/` 生成 persona
- `*.txt` WhatsApp 匯出原文
- Python cache 和虛擬環境

只保留 `exes/example_xiaomei/` 作為假資料範例。
