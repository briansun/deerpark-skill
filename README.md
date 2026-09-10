# deerpark-skill

讓你的 AI Agent（Claude Code、Codex，以及任何支援 [Agent Skills](https://agentskills.io) 標準的工具）學會查詢、閱讀、引用 **漢文大藏經**（CBETA）。

資料來源是 [deerpark.app](https://deerpark.app) 的公開 URL：4,303 部、17,862 卷、約 1.9 億字的佛經全文，加上全文檢索、佛學辭典、AI 導讀、佛經故事與精華教授。**不需要 API key，不需要登入，不需要安裝 CLI**——Agent 只要能發 HTTP GET 就能用。

```
deerpark-skill/
├── SKILL.md                  # 技能本體：站點概覽、決策樹、端點速查、引用規範
└── references/
    ├── endpoints.md          # 每個端點的完整參數與回應欄位
    ├── canons.md             # 藏經代碼、收錄範圍、分類、行號格式
    └── recipes.md            # 十個常見任務的完整指令流程
```

## 安裝

### Claude Code

個人層級（所有專案都能用）：

```bash
git clone https://github.com/briansun/deerpark-skill.git ~/.claude/skills/deerpark
```

專案層級（跟著 repo 走，適合團隊共享）：

```bash
git clone https://github.com/briansun/deerpark-skill.git .claude/skills/deerpark
```

之後在對話裡問佛經相關的問題，Claude 會自動載入這個 skill；也可以用 `/deerpark` 手動觸發。更新：進到該目錄 `git pull`。

### Codex（OpenAI Codex CLI）

```bash
git clone https://github.com/briansun/deerpark-skill.git ~/.codex/skills/deerpark
```

專案層級放 `.codex/skills/deerpark`。Codex 會在啟動時讀取 `SKILL.md` 的 `name` / `description`，需要時載入完整內容。

### 其他支援 SKILL.md 的 Agent

Agent Skills 是開放格式（一個資料夾 + `SKILL.md`），把這個 repo clone 到工具的 skills 目錄即可，例如：

| 工具 | 目錄 |
|------|------|
| Cursor | `.cursor/skills/deerpark` 或 `~/.cursor/skills/deerpark` |
| Gemini CLI | `.gemini/skills/deerpark` 或 `~/.gemini/skills/deerpark` |
| GitHub Copilot CLI | `.github/skills/deerpark` 或 `~/.copilot/skills/deerpark` |

各工具的目錄可能隨版本變動，以該工具的文件為準。

### 不支援 skills 的 Agent / 自己寫的 Agent

`SKILL.md` 就是一份寫給 LLM 看的說明書。把它的內容貼進 system prompt、`AGENTS.md`、`CLAUDE.md` 或工具描述裡即可；需要細節時再把 `references/` 裡對應的檔案一起給它。

單檔下載：

```bash
curl -fsSL https://raw.githubusercontent.com/briansun/deerpark-skill/main/SKILL.md
```

## 使用案例

安裝後直接用自然語言問，Agent 會自己決定打哪些 URL。

**找出處**

> 「凡所有相皆是虛妄」是哪部經說的？前後文是什麼？

Agent 會先查精華教授或直接在《金剛經》內全文檢索，回你：出自《金剛般若波羅蜜經》（後秦鳩摩羅什譯，CBETA T0235），附原句、上下文與 https://deerpark.app/cbeta/T0235 。

**認識一部經**

> 《楞嚴經》講什麼？有幾卷？誰譯的？我該從哪一卷開始讀？

Agent 用 `/api/v1/work/T0945` 拿到導讀、譯者、分卷，配合 `/api/v1/toc/T0945` 的章品目錄給你建議。

**讀原文並解釋**

> 幫我讀《法華經》的〈普門品〉，逐段用白話解釋。

Agent 從目錄找到普門品在第 7 卷，用 `/api/v1/text/T0262/7` 取 Markdown 純文字（不是 100 KB 的 HTML），逐段翻譯。

**查名相**

> 「阿賴耶識」和「如來藏」有什麼關係？

Agent 查五部佛學辭典（`/api/v1/dict/lookup/…`），再用全文檢索找《成唯識論》《楞伽經》等原文佐證。

**主題研究**

> 比較《雜阿含經》《金剛經》《成唯識論》對「無我」的說法，每部各引兩段原文。

Agent 在三部經內分別檢索，引文附卷數與 URL。

**找故事**

> 找幾個適合講給小孩聽的本生故事，附原典出處。

Agent 用 `/api/stories?category=本生&search=…`，每則故事帶經名、卷數，可回到原文。

**寫作與引用**

> 我在寫一篇關於「布施」的文章，幫我找 5 段經文，用學術格式引用。

Agent 產出「《經題》，譯者，CBETA Txxxx, 卷 n, p. 749a12」格式的引用，行號從 HTML 版取得。

**下載**

> 把《地藏經》的 ePub 給我。

Agent 從 `/api/v1/download/epub/T0412` 拿到 CBETA 官方檔案連結。

## 直接用 URL（不透過 Agent）

所有端點都可以在瀏覽器或 `curl` 裡直接打，完整文件見 https://deerpark.app/apidocs ：

```bash
curl "https://deerpark.app/api/search/title?q=心經"           # 經題搜索
curl  https://deerpark.app/api/v1/work/T0251                   # 一部經的資訊與導讀
curl  https://deerpark.app/api/v1/text/T0251/1                 # 經文 Markdown
curl "https://deerpark.app/api/v1/fts/works/色即是空"           # 哪些經包含這句
curl  https://deerpark.app/api/v1/dict/lookup/般若              # 佛學辭典
```

## 使用規範

- 經文來自 [CBETA](https://cbeta.org)（CC BY-NC-SA 3.0），deerpark 的導讀、故事、精華、專欄為 CC BY-NC-SA 4.0。請在輸出中保留出處與連結。
- 全文檢索走資料庫，請節制：一次任務數次查詢即可，不要用迴圈掃全藏。經目、導讀、目錄、經文內容都有 CDN 快取，可以放心用。
- 導讀、故事、精華是 AI 生成的現代解讀，不是經文本身；引用時請區分。

## 回報問題

端點行為、skill 內容有誤，或希望增加新的資料入口，請開 [issue](https://github.com/briansun/deerpark-skill/issues)。

## License

MIT
