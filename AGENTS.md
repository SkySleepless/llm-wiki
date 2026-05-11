# LLM Wiki Agent — 通用個人知識庫

本文件定義 Agent 如何維護、查詢與審計知識庫。**完全由 Agent 自主執行**。

---

## 起源

本專案實現 Andrej Karpathy 的 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式——其核心思想是：與其每次查詢時透過 RAG 從原始文件中重新推導知識，不如讓 LLM 逐步構建並維護一個持久、結構化的維基，讓知識隨著時間複利累積。

---

## 架構

四個層次：

| 層次 | 內容 | 撰寫者 |
| :--- | :--- | :--- |
| `raw/` | 不可變的原始文件（文章、書籍、章節、筆記、逐字稿、標註、圖片） | 你 |
| `wiki/` | 結構化的 Markdown 頁面（概念、作者、辯論、綜合、來源筆記、專案、領域索引） | Agent |
| `schema` | 本文件（操作指令）、`index.md`（主中心）、`domains/*.md`（領域索引）、`log.md`（變更日誌）、`overview.md`（領域摘要） | 你 + Agent |
| `graph/` | 自動生成的知識圖譜（`graph.json` + `graph.html` + `graph-report.md`） | Agent |

索引採用 **4 層分級設計**，確保查詢成本恆定，不隨頁面數量增長而膨脹：

```
L0: wiki/index.md           主中心（~2-3K chars）
    └── 領域集群表 → 指向各領域索引

L1: wiki/domains/{domain}.md  領域索引（~5-15K each）
    ├── 中醫學、易經、丹道、文學、兵家、道家、命理、科技、財經、經濟、一般
    └── 列出該領域所有 sources / concepts / entities

L2: wiki/sources/, concepts/, entities/  獨立知識頁面（現有）
L3: wiki/syntheses/                      跨領域綜合頁面（現有）
```

---

## 目錄結構

```
raw/                  # 原始文件 — 永不修改
  articles/           # 文章
  books/              # 書籍
  chapters/           # 章節
  notes/              # 筆記
  annotations/        # 標註
  transcripts/        # 逐字稿
  images/             # 圖片（可供 wiki 引用）
  _staging/           # 唯一入口：所有待攝入文件放此處，daemon 每日 03:00 自動處理
    .done/            # ingest 成功後原始檔搬入此處（按日期歸檔）
  web-query/          # query.py 自動下載的網路搜索結果（直接 ingest，不走 _staging/）
wiki/                 # Agent 完全擁有此層
  index.md            # 主索引（集群式組織，見下方「導航設計」）
  log.md              # 僅追加的時序記錄
  overview.md         # 跨領域綜合摘要（動態合成）
  domains/            # 領域索引頁面（由 update_index.py 自動生成，每領域一個檔案）
  sources/            # 來源筆記：每份原始文件的摘要頁。深度攝入模式下含子目錄（sources/{slug}/）存放章節頁面
  entities/           # 人物、組織、地點、產品、事件
  concepts/           # 觀念、方法、理論、術語
  authors/            # 思想家／作者頁面
  debates/            # 框架化的思想辯論
  syntheses/          # 綜合頁面：跨來源的論證性概述，以及曾回答過的問題
  projects/           # 活躍的研究或寫作專案
finance_report/       # 每日財經新聞報告（自動收集，不經 LLM 處理）
outputs/              # 產出
  essays/             # 論文
  slides/             # 簡報
  handouts/           # 講義
  tables/             # 表格
  reports/            # 自動報告
    daily/               # 每日跨領域研究摘要
    finance-swot/        # 每日財經 SWOT 分析（07:00 自動產生）
    stock-recommendations/  # 每日股票推薦 + 週末策略回顧
    qimen-market/        # 每日奇門遁甲市場評估
    post-market/         # 每日收盤後更新（收盤價 + 消息面校正）
conversations/        # 對話記錄存檔
archive/              # 歸檔
graph/                # 自動生成的圖譜資料
  graph.json          # 節點與邊的資料
  graph.html          # 互動式 vis.js 視覺化
  graph-report.md     # 圖譜健康報告
.opencode/
  skills/
    llama-server/     # llama.cpp qwen3 + DeepSeek flash 技能
    web-search/       # DuckDuckGo + Wikipedia 搜索技能
    image-search/     # 圖片搜索技能
  commands/
    ingest, query, health, lint, graph, ask, update-index, remove-sources, maintain, daemon, propagate-unverified
src/                  # 獨立 Python 腳本
  llm.py              # 統一 LLM 客戶端（qwen3 + DeepSeek flash，自動升級）
  search.py           # Web 搜索、圖片搜索
  domain.py           # 領域偵測與提示詞優化
  wiki_io.py          # Wiki 檔案 I/O（read/write/slugify）
  health.py           # 結構完整性檢查（無 LLM 呼叫）
  lint.py             # 內容品質檢查（使用 LLM + 圖譜感知）
  graph.py            # 知識圖譜生成
  ingest.py           # 攝入工作流
  query.py            # 查詢工作流（auto-save + web search + images + fact-check）
  factcheck.py        # 事實核查 pipeline：wiki + web + qwen3
  maintain.py         # 定時維護腳本
  daemon.py           # 背景守護行程
  scrape_finance.py   # 每日財經新聞爬蟲（每小時）+ SWOT 分析 + 股票推薦（馬爾可夫鏈 + 張量運算）+ 奇門遁甲驗證
  scrape_daily.py     # 每日跨領域研究文獻收集（摘要報告）
  qimen.py            # 奇門遁甲排盤與金融解讀模組（干支曆法、陰陽遁、九宮排盤）
  stock_analysis.py   # 股票分析模組（馬爾可夫鏈 + 張量運算 + 技術指標評分）
  tools/              # 工具腳本（refinement_loop, update_index, heal, pdf2md, verify, clean_raw 等）
```

---

## 程式架構

所有工作流程採用 **Pipeline 架構**：每個 `src/*.py` 包含一系列以 `step_` 開頭的步驟函式，依序串接執行，資料透過 `context dict` 在步驟間傳遞。

```
context = {"input": ..., "state": {}}
context = step1_init(context)
context = step2_process(context)
context = step3_output(context)
```

核心檔案對應的工作流程：

| 檔案 | Pipeline 步驟 |
|------|--------------|
| `ingest.py` | parse → convert → read → split → L1_master → L2_sections → L3_synthesis → write_all → update_metadata → validate → report<br>**上下文載入**：讀取 `index.md`（L0 主中心）→ 偵測來源領域 → 載入 `domains/{domain}.md`（L1 領域索引）→ 載入最近來源頁面 |
| `query.py` | read_index → identify_cluster → read_synthesis → find_pages → refine_query → search_web → search_images → assemble_sources → generate_answer → extract_claims → factcheck_claims → print → save |
| `lint.py` | check_orphans → check_broken_links → check_contradictions → check_stale → check_missing_entities → check_link_density → check_images → check_unverified → check_weak_sources → graph_checks → semantic_lint → compose_report |
| `build_graph.py` | extract_wikilinks → infer_relationships → detect_communities → save_graph → save_html → generate_report |
| `scrape_finance.py` | scrape_headlines → dedup → fetch_content → save → swot_analysis → (extract_news_signals → post_market_update) → qimen_analysis → stock_recommendations → (weekend_review) |
| `qimen.py` | calculate_ganzhi → determine_solar_term → get_yin_yang_dun → pai_pan → analyze_finance |
| `stock_analysis.py` | fetch_stock_data → build_markov_chain → build_tensor_analysis → compute_technical_score → generate_recommendation |

---

## 頁面類型與模板

所有頁面使用 YAML frontmatter：

### 通用 YAML Frontmatter

```yaml
---
title: "頁面標題"
type: source | entity | concept | author | debate | synthesis | project
tags: []              # 例如：[中醫, 周易]
sources: []           # 引用自哪些 source slug
related: []           # 3–5 個最接近的鄰居頁面（by filename stem）
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---
```

### 來源筆記頁面（`wiki/sources/`）

```markdown
---
title: "來源標題"
type: source
tags: [領域標籤]
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 摘要
## 關鍵主張
## 重要引文
## 相關頁面
```

### 概念頁面（`wiki/concepts/`）

```markdown
---
title: "概念名稱"
type: concept
tags: []
sources: []
related: []
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 定義
## 關鍵思想家
## 相關概念
## 實踐方法  ← 必須包含，說明具體的學習或應用步驟
## 來源支撐
```

### 作者頁面（`wiki/authors/`）

```markdown
---
title: "作者名"
type: author
tags: []
related: []
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 簡介
## 主要著作
## 核心概念
## 相關性
```

### 辯論頁面（`wiki/debates/`）

```markdown
---
title: "辯論主題"
type: debate
tags: []
sources: []
related: []
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 立場
## 關鍵文本
## 當前狀態
```

### 綜合頁面（`wiki/syntheses/`）

```markdown
---
title: "綜合主題"
type: synthesis
tags: []
sources: []
related: []
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 論證性概述
## 跨來源分析
## 結論與開放問題
```

### 專案頁面（`wiki/projects/`）

```markdown
---
title: "專案名稱"
type: project
tags: []
sources: []
related: []
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
---

## 目標
## 核心概念
## 相關來源
## 產出
```

內文使用 `[[PageName]]` 鏈接到其他 wiki 頁面。  
可使用 `![alt](../raw/images/filename.jpg)` 或 `![alt](url)` 嵌入圖片。  
**語言**：繁體中文（除程式碼、引用原文外）。所有產出（ingest / query / lint / daily report / SWOT / graph report / heal）皆透過 OpenCC `s2tw` 強制轉換為正體中文。

---

## 導航設計

約 10 個來源和 50 多個頁面後，扁平的字母順序索引會變得難以瀏覽。系統使用**四層索引串聯**，在維基增長時保持查詢成本恆定：

### 1. 主中心（`wiki/index.md` — L0）

`wiki/index.md` 不再是扁平列表，而是緊湊的領域樞紐。僅包含：
- 摘要段落
- 領域集群表（領域名 → `domains/{domain}.md` + 核心頁面）
- 跨領域概念集群（簡述）

大小控制在 2–3K chars，永遠適合 LLM 上下文視窗。

### 2. 領域索引（`wiki/domains/{domain}.md` — L1）

每個領域有獨立的索引頁面，列出該領域內所有 sources / concepts / entities / syntheses。範例：
```markdown
# 中醫學 — Domain Index

## Sources (42)
- [黃帝內經](sources/huangdi-neijing.md) — 中醫理論體系奠基經典
- [金匱要略](sources/jin-gui-yao-lue-fang-lun.md) — 張仲景雜病論治
...

## Concepts (98)
- [陰陽](concepts/陰陽.md) — 對立統一的哲學觀念，中醫基礎理論核心
...

## Entities (46)
- [張仲景](entities/張仲景.md) — 東漢醫學家，傷寒雜病論作者
...
```

查詢時先識別領域，再載入該領域索引搜尋相關頁面。每個領域索引約 5–15K chars，單獨載入適合上下文視窗。

### 3. 綜合頁面（`wiki/syntheses/` — L3）

綜合頁面是維基的「深層溝槽」：跨越一組相關頁面的、預先消化的論證性概述。當某個集群存在綜合頁面時，查詢會先讀取它——一個頁面代替六個。

### 4. `related:` YAML 欄位

每個概念和作者頁面都帶有 `related:` frontmatter 欄位，列出 3 到 5 個最接近的鄰居。閱讀完一個頁面後，Agent 使用 `related:` 導航到下一個最相關的頁面，無需重新掃描索引。

**串聯流程：** 主中心 → 領域索引 → 綜合 → 相關欄位 → 獨立頁面。

---

## 使用方式

用日常語言描述需求，或使用命令：

### 核心命令

| 命令 | 對應工作流程 | 說明 |
|------|-------------|------|
| `ingest <file>` | 攝入工作流 | 攝入單個來源 |
| `query: <question>` | 查詢工作流 | 研究查詢，自動存為 synthesis |
| `health` | 健康檢查 | 結構完整性，無 LLM 呼叫 |
| `lint` | 深層審計 | 內容品質檢查 |
| `build graph` | 知識圖譜生成 | 生成視覺化圖譜 |

### 輔助命令

| 命令 | 說明 |
|------|------|
| `/ask <prompt>` | 快速問 LLM（ad-hoc） |
| `/update-index` | 同步 index.md 與實際檔案 |
| `/remove-sources` | 移除來源及相關 wiki 頁面 |
| `/maintain` | 定時維護（health + index + graph + lint） |
| `/daemon` | 背景守護行程 |
| `/propagate-unverified` | 將引用待確認來源的具體語句也標為待確認 |
| `/factcheck` | 事實核查（claim → wiki + web + qwen3 → verdict） |
| `/refine <concept>` | 提煉特定概念 |
| `/synthesize --cluster <name>` | 為集群建立綜合頁面 |
| `/audit-refinement` | 煉化專用審計 |
| `/refinement-loop <cycles>` | 自動執行完整煉化迴圈 |

---

## LLM 後端

| 模型 | 位置 | 上下文 | 輸出上限 | 觸發方式 |
|------|------|--------|---------|---------|
| qwen3 (llama.cpp) | `http://127.0.0.1:8080` | 32K tokens | 32K | 預設 |
| DeepSeek v4 flash | `https://api.deepseek.com` | **1M tokens** | **384K** | `--flash` 或自動（prompt > 32K / qwen3 無回應） |

**自動升級**：當 prompt 超過 qwen3 的 32K 限制，或 qwen3 伺服器無回應時，自動切換至 DeepSeek flash。

**環境變數**：`DEEPSEEK_API_KEY` — DeepSeek API 金鑰

---

## 領域感知系統

| 領域 | 查詢重點 | 攝入重點 | 審計重點 |
|------|---------|---------|---------|
| 🏥 中醫學 | 引用經典原文、辨證要點、方劑組成 | 提取醫學理論、臨床症狀 | 檢查術語準確性、經典引用 |
| 📖 易經 | 卦象結構、卦辭彖傳、卦序演進 | 提取卦象哲學、關聯分析 | 檢查卦序、卦辭正確性 |
| ☯️ 丹道 | 功法步驟、火候要點、丹經原文 | 提取丹法理論、修煉步驟 | 檢查術語、流派區分 |
| 🐵 文學 | 角色分析、情節脈絡、原文佐證 | 提取故事梗概、角色關係 | 檢查角色、情節正確性 |
| ⚔️ 兵家 | 原文引用、歷史戰例、戰略原則 | 提取戰略原則、歷史案例 | 檢查原文、案例真實性 |
| 🏛️ 道家 | 經典引用、哲學概念、學派對比 | 提取哲學主張、關鍵原文 | 檢查概念定義、原文引用 |
| 🔮 命理 | 命盤分析、星曜特性、案例說明 | 提取命理規則、星曜屬性 | 檢查術語、判斷邏輯 |
| 🎬 娛樂 | 基本資料、近期動態、可靠來源 | 提取藝人經歷、作品 | 檢查資訊正確性 |
| 🤖 科技 | 技術名稱、版本號、協議規範 | 提取技術架構、系統設計 | 檢查技術術語準確性 |
| 💰 經濟 | 經濟指標、政策背景、數據來源 | 提取經濟數據、產業趨勢 | 檢查數據可驗證性 |

查詢時自動偵測問題所屬領域，注入對應的領域優化提示詞。

---

## 攝入工作流（Ingest）

觸發方式：`ingest <file>` 或 `python src/ingest.py <file>`

**支援格式**：`.md` 直接讀取。非 markdown 檔案會透過 [markitdown](https://github.com/microsoft/markitdown) 自動轉為 markdown 再處理。使用 `--no-convert` 可跳過自動轉換。

### 統一的 3 層攝入管道

所有文件均使用相同的 3 層管道，無論大小：

```
原始文件
    ↓ 語義切片（markdown 標頭 → 語義邊界 → 段落邊界，重疊 15%）
L1：主頁面 (Master)
    → wiki/sources/{slug}.md
    → 全文概述 + 目錄 + 關鍵主題 + 導航樞紐
    ↓ 逐段處理（每段獨立 LLM 呼叫，含母文檔上下文）
L2：章節頁面 (Sections)
    → wiki/sources/{slug}/01-name.md ... 0N-name.md
    → 每段深度提取：摘要、關鍵主張、引文、概念、實體
    ↓ 全段綜合（一次 LLM 呼叫）
L3：概念合成 (Synthesis)
    → wiki/concepts/X.md, entities/Y.md, authors/Z.md, debates/W.md
    → 跨段概念提取、定義、實體、作者、辯論（含 ## 實踐方法）
```

**目錄結構範例**：
```
wiki/
  sources/
    周易.md                    # L1：主頁面
    周易/                      # L2：章節目錄
      01-shang-jing.md
      02-xia-jing.md
  concepts/
    乾卦.md                    # L3：概念頁面（跨段合成）
    ...
```

**特性**：
- 語義切片：Trigram 相似度檢測主題變化點作為自然邊界，避免語境斷裂
- 15% 重疊區間：相鄰切片保留上下文，防止關鍵資訊在邊界處遺失
- 母文檔上下文：L2 章節處理時包含 L1 主頁面的 Overview 作為母文檔背景（Parent Document Retrieval）
- 預設使用 qwen3（32K 上下文），超大提示自動升級至 DeepSeek Flash
- `--flash` 可強制所有層級使用 Flash

**LLM 後端**：預設 qwen3（免費）；`--flash` 強制 DeepSeek v4 flash；若 qwen3 無法連接則自動切換至 Flash。

**參數**：
- `--flash` — 使用 DeepSeek v4 flash
- `--no-convert` — 跳過非 markdown 檔案的自動轉換
- `--from-staging` — 掃描 `raw/_staging/` 中的檔案，批次攝入後搬移至 `.done/`

---

## 查詢工作流（Query）

觸發方式：`query: <question>` 或 `python src/query.py "<question>"`

**流程**（四層導航串聯）：

1. **識別領域**：從 `wiki/index.md`（L0 主中心）找出問題所屬的領域
2. **載入領域索引**：讀取 `wiki/domains/{domain}.md`（L1）找出相關頁面
3. **讀取綜合頁面**：若該領域存在綜合頁面（`syntheses/`），優先讀取
4. **跟隨相關欄位**：使用 `related:` 欄位導航到相鄰頁面
4. 若頁面不足，自動掃描 `raw/` 中檔名匹配的原始文件 → **自動攝入**後重新查詢
5. 再掃描 `sources/` 中內容匹配的來源頁面 → 加入上下文
6. LLM 優化搜索查詢語句
7. 搜索 DuckDuckGo + Wikipedia 補充資訊
8. LLM 判斷是否需要圖片 → 搜索圖片
9. 全部資訊（wiki + web + images）一次送給 LLM 綜合回答
10. 自動存為 `wiki/syntheses/<slug>.md` 並更新 index/log
11. **事實核查**：自動提取答案中的關鍵聲明 → 搜索 wiki + 搜索 web → qwen3 分析 → 附上**信心指數**

**參數**：
- `--no-save` — 跳過自動存檔
- `--flash` — 使用 DeepSeek v4 flash

---

## 健康檢查工作流（Health）

觸發方式：`health` 或 `python src/health.py`

**純結構檢查 — 不呼叫 LLM**：
- **空／樁檔案**：無實質內容的頁面
- **索引同步**：`index.md` 與 `domains/*.md` 與實際檔案是否一致（多檔案驗證）
- **日誌覆蓋**：來源頁面是否缺少 ingest 日誌（含子目錄掃描）
- **層級結構**：主頁面 TOC 與章節檔案是否一致、章節頁面是否鏈接回主頁面、孤立章節目錄

**參數**：
- `--save` — 儲存至 `wiki/health-report.md`
- `--json` — 機器可讀輸出

---

## 深層審計工作流（Lint）

觸發方式：`lint` 或 `python src/lint.py --save`

**檢查項目**：
- **孤立頁面**：無入向 `[[wikilinks]]` 的頁面
- **斷裂鏈接**：指向不存在頁面的 `[[WikiLinks]]`
- **矛盾**：不同頁面間的衝突陳述
- **過時摘要**：有新來源但未更新的頁面
- **遺漏實體**：被 3 個以上頁面提及但無獨立頁面
- **稀疏頁面**：出向鏈接少於 2 個
- **資料缺口**：建議補充新來源
- **圖片檢查**：標記指向不存在檔案的 `![alt](url)`
- **待確認語句**：列出含有 `[待確認]` 標記的頁面與數量
- **薄弱的來源支撐**：概念頁面引用來源過少（< 2）

**圖譜感知檢查**（需要 `graph.json`）：
- **樞紐樁**：度數 > μ+2σ 但內容單薄
- **脆弱橋樑**：社群間單一邊連接
- **孤立社群**：無對外連接的頁面群

**3 層結構檢查**（自動，無需 `--refinement`）：
- **章節樁**：章節頁面內容過少（< 300 字）或缺少摘要/主張章節
- **層級覆蓋**：主頁面 TOC 斷鏈（L1→L2）、章節頁面缺少概念鏈接（L2→L3）、章節頁面未被概念頁引用（L3→L2）

**煉化專用檢查**（由 `lint --refinement` 觸發）：
- **萃取完整性**：`raw/` 中的文件是否都有對應的 `sources/` 頁面（支援 `sources/{slug}/` 子目錄）？
- **提煉覆蓋率**：被 3+ 個來源提及的概念是否都有獨立頁面？
- **合成必要性**：是否有集群達到 4+ 來源但尚未創建綜合頁面（含章節頁面）？
- **定義清晰度**：概念頁面的「定義」章節是否過短（< 50 字）或過於抽象？
- **實踐方法缺失**：概念頁面是否缺少「實踐方法」章節？
- **綜合過時**：綜合頁面引用來源中有新攝入的來源未被納入。

結果以優先問題列表呈現。

**參數**：
- `--save` — 儲存報告至 `wiki/lint-report.md`
- `--flash` — 使用 DeepSeek v4 flash 分析
- `--refinement` — 啟用煉化專用檢查項目

### Health 與 Lint 的分界

| 維度 | `health` | `lint` |
|------|----------|--------|
| 範圍 | 結構完整性 | 內容品質 |
| LLM 呼叫 | 無 | 有（語義分析） |
| 成本 | 免費 | 消耗 token |
| 頻率 | 每次會話 | 每 10–15 次 ingest |
| 檢查 | 空檔案、索引、日誌、3 層結構 | 孤立頁、斷鏈、矛盾、缺口、圖片、章節樁、層級覆蓋 |
| 工具 | `src/health.py` | `src/lint.py` |
| 執行順序 | 優先（pre‑flight） | 健康檢查通過之後 |

---

## 圖譜工作流（Graph）

觸發方式：`build graph` 或 `python src/build_graph.py`

**流程**：
1. 掃描所有 wiki 頁面中的 `[[wikilinks]]`
2. 建立節點（每頁一個）與邊（每條鏈接一個）
3. 推論隱含關係（`INFERRED` / `AMBIGUOUS`）
4. Louvain 社群檢測
5. 寫入 `graph/graph.json` + `graph/graph.html`（vis.js 互動視覺化）

**參數**：
- `--no-infer` — 跳過語義推論（更快）
- `--report` — 生成圖譜健康報告
- `--save` — 儲存報告至 `graph/graph-report.md`
- `--open` — 在瀏覽器中開啟圖譜
- `--clean` — 重新推論（清除快取）
- `--flash` — 使用 DeepSeek v4 flash 推論

---

## 定時維護工作流（Maintain）

觸發方式：`/maintain` 或 `python src/maintain.py`

自動執行：health → update-index → build_graph → lint

**Cron 排程建議**（現已由 daemon 自動處理，以下僅供參照）：
```
# 每日 3am：煉化迴圈（由 daemon.py 03:00 自動執行）
0 3 * * * cd /path/to/repo && python3 src/tools/refinement_loop.py --cycles 1 --flash >> wiki/cron.log 2>&1

# 每週日 4am：完整檢查（含 lint + DeepSeek）
0 4 * * 0 cd /path/to/repo && python3 src/maintain.py --flash >> wiki/cron.log 2>&1
```

**參數**：
- `--quick` — 跳過 lint（節省 token）
- `--flash` — Lint 使用 DeepSeek v4 flash

### 背景守護行程

```bash
python src/daemon.py start     # 啟動守護行程（每24小時自動維護）
python src/daemon.py stop      # 停止
python src/daemon.py status    # 查看狀態與最近日誌
python src/daemon.py logs      # 即時查看日誌
```

守護行程每 24 小時執行一次 `maintain.py --auto`（一般日 health + index + graph，週日含 lint）。
守護行程每日 03:00 執行 `refinement_loop.py --cycles 1 --flash`，進行煉化迴圈（萃取 _staging/ 新檔 → 補概念頁 → 補綜合頁）。
守護行程每日 07:00 執行 `scrape_finance.py --swot`，根據過去 24 小時財經新聞進行 SWOT 分析（強弱危機分析），存入 `outputs/reports/finance-swot/`。
守護行程每日 08:00 執行 `scrape_finance.py --recommend --qimen`，產生股票買入／賣出／持倉推薦（馬爾可夫鏈 + 張量運算），並附奇門遁甲交叉驗證，存入 `outputs/reports/stock-recommendations/`。
守護行程每日 08:05 執行 `scrape_finance.py --qimen`，獨立產出奇門遁甲市場評估報告，存入 `outputs/reports/qimen-market/`。
守護行程每週六／日 10:00 執行 `scrape_finance.py --weekend`，進行週末策略回顧：回顧本週股票表現、以長期馬爾可夫鏈（120 日）調整下週策略，並由 LLM 綜合奇門遁甲信號與量化信號提出建議。
守護行程每日 16:30（僅交易日）執行 `scrape_finance.py --post-market`，以當日收盤價重新分析所有股票，並提取消息面／產業／政策信號進行參數校正，存入 `outputs/reports/post-market/` 及 `outputs/reports/stock-recommendations/`。
此外，守護行程每小時自動執行 `scrape_finance.py --limit 5`（於每小時的 :30 執行），抓取新浪港股、21經濟網、證券時報等財經新聞，存入 `finance_report/YYYY-MM-DD/`。
守護行程每 24 小時執行一次 `scrape_daily.py --report`，產生 10 篇跨領域研究摘要，存入 `outputs/reports/daily/`。
守護行程每 6 小時執行一次事實核查掃描，自動驗證 `[待確認]` 頁面。
守護行程每日執行一次清理，移除 raw/ 中過期的自動轉換檔案。

---

## 股票分析模組（Stock Analysis）

### 概述

整合量化分析（馬爾可夫鏈、張量運算、技術指標）與奇門遁甲時空模型，產生港股買入／賣出／持倉建議，並在週末進行策略回顧與調整。

### 股票池（預設 13 隻港股）

| 股票 | 代號 | 行業 | 奇門五行 |
|------|------|------|----------|
| 中芯國際 | 0981.HK | 半導體 | 金 |
| 國藥控股 | 1099.HK | 醫藥 | 水 |
| 阿里巴巴-W | 9988.HK | 科技 | 木 |
| 騰訊控股 | 0700.HK | 科技 | 水 |
| 美團-W | 3690.HK | 科技 | 木 |
| 比亞迪股份 | 1211.HK | 新能源 | 火 |
| 小米集團-W | 1810.HK | 科技 | 火 |
| 快手-W | 1024.HK | 科技 | 土 |
| 京東集團-SW | 9618.HK | 科技 | 土 |
| 網易-S | 9999.HK | 科技 | 土 |
| 建設銀行 | 0939.HK | 銀行 | 土 |
| 銀河娛樂 | 0027.HK | 博彩 | 火 |
| 中國南方航空股份 | 1055.HK | 航空 | 金 |

### 分析引擎

#### 馬爾可夫鏈（Markov Chain）

- **狀態空間**：漲／跌／平（閾值 ±0.5%），或 5 態（大漲／小漲／平／小跌／大跌）
- **轉移概率矩陣**：基於 120 日歷史日收益率，Laplace 平滑（α=0.1）
- **穩態分佈**：矩陣冪迭代求特徵向量
- **預測**：當前狀態 → 最大概率下一個狀態

#### 張量運算（Tensor Computation）

- **6 維特徵張量**：價格動量、成交量比率、波動率、高低價差、5 日趨勢、14 日 RSI
- **SVD 低秩分解**：truncated rank-2 近似
- **異常檢測**：重構誤差 → 偏離均值倍數 → 異常分數（0–5）
- **趨勢強度**：第一主成分解釋方差比
- **成交量信號**：近 5 日 vs 30 日均量

#### 技術指標評分

- 均線交叉（MA5 vs MA20 vs MA50）— 權重 0.5
- RSI 14 日 — 權重 0.3
- MACD（EMA12 vs EMA26）— 權重 0.2
- 布林帶位置 — 權重 0.2
- **綜合評分**：−1（極空）到 +1（極多）

### 奇門遁甲整合

- **五行分類**：每隻股票歸入金／水／木／火／土
- **日時生剋**：日干（投資者）vs 時干（市場）
- **生門分析**：機會門所在宮位的星神組合
- **值符趨勢**：主導星對市場的影響方向
- **交叉驗證表**：量化建議 vs 奇門建議 → ✓（一致）／△（部分一致）／✗（矛盾）

### 使用方式

```bash
# 股票推薦
python src/scrape_finance.py --recommend

# 股票推薦 + 奇門遁甲交叉驗證
python src/scrape_finance.py --recommend --qimen

# 指定股票（支援部分庫內名稱）
python src/scrape_finance.py --recommend --qimen --stocks 中芯國際 國藥控股 阿里巴巴-W

# 週末策略回顧（強制模式：非週末亦可執行）
python src/scrape_finance.py --weekend --force-weekend

# 全管線（爬取 + SWOT + 推薦 + 奇門遁甲）
python src/scrape_finance.py --all

# 收盤後更新（收盤價 + 消息面 + 產業 + 政策校正）
python src/scrape_finance.py --post-market

# 全管線跳過特定步驟
python src/scrape_finance.py --all --no-swot --no-qimen
```

### 產出檔案

| 目錄 | 說明 |
|------|------|
| `outputs/reports/stock-recommendations/YYYY-MM-DD.md` | 每日推薦報告（買入／賣出／持倉表格 + 詳細分析 + 奇門交叉驗證表） |
| `outputs/reports/stock-recommendations/weekend-YYYY-MM-DD.md` | 週末回顧（本週表現 + 策略調整 + 馬爾可夫穩態 + LLM 策略建議） |
| `outputs/reports/qimen-market/YYYY-MM-DD.md` | 奇門遁甲市場評估（排盤摘要 + 九宮詳情 + 五行股票建議） |
| `outputs/reports/post-market/YYYY-MM-DD.md` | 收盤後報告（收盤價 + 消息面／產業／政策信號校正 + 重新推薦） |

### 週末回顧模式

觸發條件：週六或週日（或 `--force-weekend`）。

流程：
1. 奇門遁甲週末排盤 → 五行行業信號
2. 所有股票 3 個月歷史數據 → 長期馬爾可夫鏈
3. 統計本週回報（vs 5 日前）
4. LLM 綜合奇門信號 + 量化信號 → 下週策略建議（進攻／防守／觀望 + 板塊 + 個股調整）

---

## 煉化流程（Refinement Workflow）

煉化是將原始資料逐步轉化為持久、高品質、可行動知識的迭代閉環。不同於一次性攝入，煉化強調多輪精煉、跨來源綜合與持續審計。

### 煉化五階段

```
原始資料 (raw/_staging/)
    ↓
階段1：萃取 (Extract) → 來源筆記 (sources/)
    ↓
階段2：提煉 (Refine) → 概念/作者頁面 (concepts/, authors/)
    ↓
階段3：合成 (Synthesize) → 綜合頁面 (syntheses/)
    ↓
階段4：審計 (Audit) → 健康檢查 + 深層審計 (health, lint)
    ↓
階段5：應用 (Apply) → 查詢回答、專案產出 (query, projects/)
    ↑_______________________________________________| (迭代迴圈)
```

### 階段1：萃取（Extract）

**觸發**：`ingest --from-staging` 或 daemon 每日 03:00

**目標**：從原始文件中提取關鍵主張、引文、概念，不添加外部知識。

**產出**：`wiki/sources/<slug>.md` + 初步的實體/概念存根。

**注意**：現有 `raw/articles/`, `raw/books/`, `raw/chapters/` 等目錄中的文件不自動掃描。若要煉化舊文件，手動 `cp raw/articles/foo.md raw/_staging/` 讓它重新走一次流程。

### 階段2：提煉（Refine）

**觸發**：daemon 每日 03:00 自動檢測，或手動 `/refine <concept>`

**目標**：將多個來源中的概念提煉為獨立、定義清晰、有實踐指引的知識單元。

**產出**：`wiki/concepts/<概念>.md` 或更新現有概念頁面。

**自動檢測邏輯**：
1. 掃描所有 `wiki/sources/` 與 `wiki/concepts/` 頁面中的 `[[wikilinks]]`
2. 找出被 3+ 頁面提及但無獨立頁面的概念
3. 呼叫 LLM 建立概念頁面（含定義、關鍵思想家、相關概念、實踐方法、來源支撐）

**自動補強**：
- 檢查 `concepts/` 下缺 `## 實踐方法` 章節的舊頁面，呼叫 LLM 補上
- 檢查 `sources/` 下內容過短（< 300 字）且 `source_file` 存在、標記為「可重新攝入」的舊頁面，自動 copy 回 `_staging/` 等下次萃取

### 階段3：合成（Synthesize）

**觸發條件**：
- 某個概念集群有 4+ 個來源頁面
- 同一個問題被查詢 3+ 次
- 手動執行 `/synthesize --cluster <name>`

**目標**：產出跨越單一來源的論證性綜述，成為該集群的「首選查詢入口」。

**產出**：`wiki/syntheses/<主題>.md`

**自動檢測**：daemon 每日檢查所有集群，發現達到門檻的集群即自動建立綜合頁面。

### 階段4：審計（Audit）

由 `lint` 的煉化專用檢查涵蓋（見「深層審計工作流」章節），包括：
- 萃取完整性：`raw/` 中的文件是否都有對應的 `sources/` 頁面？
- 提煉覆蓋率：被 3+ 個來源提及的概念是否都有獨立頁面？
- 合成必要性：是否有集群達到 4+ 來源但尚未創建綜合頁面？
- 定義清晰度：概念頁面的「定義」章節是否過短（< 50 字）或過於抽象？
- 實踐方法缺失：概念頁面是否缺少「實踐方法」章節？
- 綜合過時：綜合頁面引用來源中有新攝入的來源未被納入。

### 階段5：應用（Apply）

應用是煉化流程的輸出端，也是迴圈的起點——應用中發現的不足反饋回萃取或提煉階段。

**觸發**：正常使用 `query:` 命令。

**煉化反饋機制**：當查詢發現某概念頁面無法回答查詢、或某綜合頁面資訊過時，自動記錄反饋供下一輪煉化處理。

### 與各工作流的整合

| 工作流 | 整合點 |
|--------|--------|
| `ingest.py` | 階段1：`--from-staging` 模式掃描 `_staging/` |
| `lint.py` | 階段4：煉化專用檢查項目 |
| `query.py` | 階段5：自動存檔為 synthesis |
| `daemon.py` | 每日 03:00 自動執行 |
| `refinement_loop.py` | 煉化引擎：串接階段1→2→3 |

### 煉化命令匯總

| 命令 | 說明 | 對應階段 |
|------|------|----------|
| `ingest --from-staging` | 萃取 `_staging/` 中新檔案 | 階段1 |
| `/refine <concept>` | 提煉特定概念 | 階段2 |
| `/refine --cluster <name>` | 提煉整個集群的所有概念 | 階段2 |
| `/synthesize --cluster <name>` | 為集群建立綜合頁面 | 階段3 |
| `/audit-refinement` | 煉化專用審計 | 階段4 |
| `/refinement-loop <cycles>` | 自動執行完整煉化迴圈 | 全部 |

---

## 命名慣例

| 類型 | 格式 | 範例 |
|------|------|------|
| Source slug | kebab-case | `huangdi-neijing.md` |
| Entity／Concept／Author 頁面 | TitleCase | `AlbertEinstein.md`、`孫悟空.md`、`習慣養成.md` |

---

## 索引格式（`wiki/index.md`）

```markdown
# Wiki Index

## Overview
- [Overview](overview.md) — 跨領域綜合摘要

## Domain Indexes

| Domain | Pages | Index |
|--------|-------|-------|
| 中醫學 | 186 | [中醫學](domains/中醫學.md) |
| 易經 | 42 | [易經](domains/易經.md) |
| 一般 | 155 | [一般](domains/一般.md) |

## Concept Clusters (cross-domain)
### 中醫基礎理論 → 核心: [[陰陽]], [[五行]], [[臟腑]], [[辨證論治]]
### 易經哲學 → 核心: [[六十四卦]], [[乾卦]], [[坤卦]], [[繫辭傳]]
```

---

## 日誌格式（`wiki/log.md`）

僅追加。每筆記錄格式：

`## [YYYY-MM-DD] <操作> | <標題>`

操作類型：`ingest`, `ingest (deep L1)`, `ingest (deep L3)`, `query`, `concept`, `author`, `debate`, `synthesis`, `fix`, `lint`, `graph`, `report`, `index`

---

## 通用領域規則

1. **引用與歸屬**：每個事實性陳述必須標註來源（`(source: slug.md)` 或 `🔍[url]`）。無來源的標註為 `[待驗證]`。
2. **避免幻覺**：若來源無具體資訊（如角色名稱、日期），應使用 `[待確認]` 或「未知」，不得憑空捏造。事實核查會標記可疑內容與**信心指數**。
3. **日常瑣事**：極短的筆記也應攝入並歸檔。知識庫的價值來自持續累積。
4. **矛盾與不確定性**：筆記中前後不一致的想法不應強行統一，應明確標註不同觀點與時間脈絡。
5. **跨領域連結**：鼓勵發現不同主題之間的隱喻或結構相似性，標記為 `INFERRED` 並附信心分數。
6. **煉化迴圈**：知識應經過五階段煉化（萃取→提煉→合成→審計→應用），形成持續迭代的品質提升閉環。

---

## 待確認管理

- `[待確認]`：標記未經可靠來源驗證的陳述
- `[已確認]`：已驗證的陳述
- 確認方法：`sed -i 's/\[待確認\]/[已確認]/g' wiki/sources/xxx.md`
- `lint` 自動統計每個頁面的待確認數量
- `query` 回答時明確區分「已驗證」和「待確認」的資訊來源
- `/propagate-unverified`：將引用待確認來源的頁面也標為待確認

---

## 已知問題

- **繁簡轉換**：所有產出管道已統一使用 OpenCC `s2tw` 強制轉換為正體中文。覆蓋範圍：`ingest.py`（所有頁面內容）、`query.py`（LLM 回答）、`lint.py`（審計報告）、`scrape_daily.py`（每日研究摘要）、`scrape_finance.py`（SWOT 分析 + 文章內文 + 股票推薦 + 奇門遁甲報告）、`qimen.py`（排盤輸出）、`stock_analysis.py`（推薦報告）、`graph.py`（圖譜報告 + 推論描述）、`tools/heal.py`（實體頁面）。另在 prompt 層級加上「Write in Traditional Chinese (繁體中文)」指令，形成雙重保障。
- **大型檔案**：所有文件統一使用 3 層攝入管道（L1 主頁面 → L2 章節頁面 → L3 概念合成），不再截斷。
- **`.ipynb_checkpoints`**：Jupyter 自動產生的檢查點檔案會被所有腳本忽略。
- **圖譜推論**：使用 qwen3 時的語義推論較慢（推理模型特性），建議使用 `--flash` 加速。

---

## 規則總結

- **禁止**修改 `raw/` 下的任何檔案。
- **必須**在每次 ingest 後更新 `wiki/index.md` 與 `wiki/log.md`（`update_index.py` 可自動同步）。
- **必須**使用 YAML frontmatter 並填寫 `last_updated`。
- **語言**：繁體中文（所有產出經 OpenCC `s2tw` 強制轉換）。
- **依賴**：`opencc-python-reimplemented`（所有產出管線的必要套件）。
- **對待瑣事**：片段文字也應攝入歸檔。
- **圖片**：可使用 `![alt](../raw/images/filename.jpg)` 或 `![alt](url)` 嵌入圖片。
