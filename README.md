# LLM Wiki — 個人知識庫

基於 Andrej Karpathy 的 [LLM Wiki 模式](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 構建的個人知識管理系統，由 AI Agent（[opencode](https://github.com/anomalyco/opencode)）自主維護。

## 核心概念

讓 LLM 逐步構建並維護一個持久、結構化的維基，知識隨著時間複利累積——而非每次查詢時透過 RAG 從原始文件中重新推導。

## 架構

| 層次 | 目錄 | 說明 |
| :--- | :--- | :--- |
| 原始文件 | `raw/` | 不可變的原始資料（文章、書籍、筆記等） |
| 結構化知識 | `wiki/` | Agent 生成的 Markdown 頁面（概念、來源、綜合等） |
| 導航層 | `AGENTS.md`, `wiki/index.md`, `wiki/domains/`, `wiki/log.md` | 主索引、領域索引、變更日誌 |
| 圖譜 | `graph/` | 自動生成的知識圖譜（JSON + HTML 視覺化） |

## 安裝

```bash
# 1. 安裝 Python 依賴
pip install -r requirements.txt

# 2. 安裝 opencode
# macOS / Linux:
curl -fsSL https://anomalyco.github.io/opencode/install.sh | bash
# 或透過 Homebrew:
# brew install anomalyco/tap/opencode

# 3. 設定環境變數
export DEEPSEEK_API_KEY="your-deepseek-api-key"
```

## 使用方式

在專案目錄中啟動 opencode，Agent 會自動讀取 `AGENTS.md` 中的操作指令：

```bash
opencode
```

### 核心命令

| 命令 | 說明 |
|------|------|
| `ingest <file>` | 攝入原始文件，自動生成 wiki 頁面 |
| `query: <question>` | 研究查詢，自動搜索 wiki + web，存為 synthesis |
| `health` | 結構完整性檢查（無 LLM 呼叫） |
| `lint` | 內容品質深層審計（LLM 驅動） |
| `build graph` | 生成互動式知識圖譜 |
| `/maintain` | 定時維護（health + index + graph + lint） |

### 輔助命令

| 命令 | 說明 |
|------|------|
| `/ask <prompt>` | 快速問 LLM |
| `/update-index` | 同步索引與實際檔案 |
| `/refine <concept>` | 提煉特定概念頁面 |
| `/factcheck` | 事實核查 |
| `/daemon` | 背景守護行程（自動維護 + 財經新聞） |

## 目錄結構

```
raw/                  # 原始文件 — 永不修改
wiki/                 # Agent 完全擁有此層
  index.md            # L0 主索引中心
  domains/            # L1 領域索引
  sources/            # L2 來源筆記
  concepts/           # L2 概念頁面
  entities/           # L2 實體頁面
  syntheses/          # L3 跨領域綜合
graph/                # 自動生成的圖譜
outputs/              # 產出（報告、論文、簡報）
src/                  # Python 腳本（Agent 呼叫用）
```

## 依賴

- Python 3.10+
- [opencode](https://github.com/anomalyco/opencode) — AI Agent 終端
- [llama.cpp](https://github.com/ggerganov/llama.cpp) — 本地 LLM 推理（qwen3, 可選）
- [DeepSeek API](https://api.deepseek.com) — 遠端 LLM（超大上下文時自動使用）

## 許可

MIT
