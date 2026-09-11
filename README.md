<p align="center">
  <strong>deerpark-skill</strong><br>
  讓 AI Agent 讀懂漢文大藏經
</p>

<p align="center">
  一個 <code>SKILL.md</code>，Claude Code、Codex、Cursor、Gemini CLI 等 agent 就能透過 <a href="https://deerpark.app">deerpark.app</a> 的公開 URL<br>
  查經、讀經、找出處、查名相、引用原文。不用 API key，不用登入，不用裝 CLI。
</p>

<p align="center">
  <a href="https://github.com/briansun/deerpark-skill/stargazers"><img src="https://img.shields.io/github/stars/briansun/deerpark-skill?style=flat&color=yellow" alt="Stars"></a>
  <a href="https://deerpark.app/apidocs"><img src="https://img.shields.io/badge/data-4%2C303_sutras_%C2%B7_190M_chars-orange?style=flat" alt="4,303 sutras"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-MIT-green?style=flat" alt="MIT"></a>
  <a href="https://skills.sh/briansun/deerpark-skill"><img src="https://skills.sh/b/briansun/deerpark-skill" alt="skills.sh"></a>
</p>

<p align="center">
  <a href="#看它怎麼答">看它怎麼答</a> ·
  <a href="#安裝">安裝</a> ·
  <a href="#使用案例">使用案例</a> ·
  <a href="#裡面有什麼">裡面有什麼</a> ·
  <a href="#直接用-url">直接用 URL</a> ·
  <a href="#使用規範">使用規範</a> ·
  <a href="#給-agent-的安裝指引">給 agent 的安裝指引</a>
</p>

---

> **English**: An [Agent Skill](https://agentskills.io) that teaches Claude Code, Codex and other agents to search, read and cite the Chinese Buddhist Canon (CBETA) through the public URLs of deerpark.app — 4,303 works, 17,870 fascicles, ~190M characters of Classical Chinese, plus full-text search, Buddhist dictionaries, AI summaries, stories and highlighted passages. Every essay on the [deerpark.app blog](https://deerpark.app/blog) was written with this skill. To install, tell your agent: *"Install the skill from https://github.com/briansun/deerpark-skill"* (or run `npx skills add briansun/deerpark-skill -g`).

> **實戰證明**：[deerpark.app 專欄](https://deerpark.app/blog)上的每一篇文章（讀經筆記、專題研究、祖師傳記）都是用這個 skill 寫出來的——查經、找出處、引原文，全部走這些 URL。

## 看它怎麼答

<table>
<tr>
<th width="50%">沒裝 skill</th>
<th width="50%">裝了 skill</th>
</tr>
<tr>
<td valign="top">

> **你**：「應無所住而生其心」出自哪部經？
>
> **Agent**：這句話出自《金剛經》，是佛陀對須菩提說的……（憑記憶作答，沒有原文，沒有出處連結，偶爾會把卷數或譯者記錯）

</td>
<td valign="top">

> **你**：「應無所住而生其心」出自哪部經？
>
> **Agent**：出自《金剛般若波羅蜜經》，後秦鳩摩羅什譯（CBETA T0235），全經一卷。原文：
> 「是故須菩提，諸菩薩摩訶薩應如是生清淨心，不應住色生心，不應住聲、香、味、觸、法生心，**應無所住而生其心**。」
> 閱讀：https://deerpark.app/cbeta/T0235
>
> *（背後打了 2 個 URL：精華教授搜索 → 經內全文檢索）*

</td>
</tr>
</table>

Agent 的回答來自線上經文，不是記憶；每個結論都附可點的出處。

## 安裝

打開你的 agent（Claude Code、Codex、Cursor……），把這句話貼給它：

```
請幫我安裝 skill：https://github.com/briansun/deerpark-skill
```

就這樣。agent 會自己把 repo 放到它的 skills 目錄。裝完問一個佛經問題，或輸入 `/deerpark`，看它是否開始打 deerpark.app 的 URL。

<details>
<summary>想自己動手</summary>

<br>

```bash
# 用 skills CLI（自動偵測機器上的 agent；-g 裝到使用者目錄）
npx skills add briansun/deerpark-skill -g

# 或直接 clone 到 agent 的 skills 目錄，例如 Claude Code / Codex
git clone https://github.com/briansun/deerpark-skill.git ~/.claude/skills/deerpark
git clone https://github.com/briansun/deerpark-skill.git ~/.codex/skills/deerpark
```

其他 agent 的目錄見 skills CLI 的 [Supported Agents](https://github.com/vercel-labs/skills#supported-agents)。不支援 skills 的 agent，把 [SKILL.md](./SKILL.md) 的內容貼進 system prompt 或 `AGENTS.md` 即可。

</details>

## 使用案例

裝好後直接用自然語言問，agent 會自己決定打哪些 URL。

| 你問 | Agent 做什麼 |
|---|---|
| 「凡所有相皆是虛妄」是哪部經說的？前後文是什麼？ | 查精華教授或在《金剛經》內全文檢索，回你經題、譯者、CBETA T0235、原句上下文與連結 |
| 《楞嚴經》講什麼？有幾卷？我該從哪一卷開始讀？ | 用 `/api/v1/work/T0945` 拿導讀、譯者、分卷，配合目錄給建議 |
| 幫我讀《法華經》的〈普門品〉，逐段用白話解釋 | 從目錄找到普門品在第 7 卷，用 `/api/v1/text/T0262/7` 取純文字（不是 100 KB 的 HTML）逐段翻譯 |
| 「阿賴耶識」和「如來藏」有什麼關係？ | 查五部佛學辭典，再用全文檢索找《成唯識論》《楞伽經》原文佐證 |
| 比較《雜阿含經》《金剛經》《成唯識論》對「無我」的說法，各引兩段 | 在三部經內分別檢索，引文附卷數與 URL |
| 找幾個適合講給小孩聽的本生故事，附出處 | `/api/stories?category=本生`，每則帶經名卷數，可回到原文 |
| 我在寫一篇關於「布施」的文章，幫我找 5 段經文，用學術格式引用 | 產出「《經題》，譯者，CBETA T08, no. 235, p. 749a12」格式，行號從 HTML 版取得 |
| 把《地藏經》的 ePub 給我 | 從 `/api/v1/download/epub/T0412` 拿到 CBETA 官方檔案連結 |

## 裡面有什麼

```
deerpark-skill/
├── SKILL.md                  # 技能本體（~160 行）：站點概覽、決策樹、端點速查、引用格式、備援路徑
└── references/               # agent 需要時才載入
    ├── endpoints.md          # 每個端點的完整參數與回應欄位
    ├── canons.md             # 藏經代碼、收錄範圍、分類、行號格式、朝代異名
    └── recipes.md            # 十個常見任務的完整指令流程
```

資料範圍：大正藏 T、卍續藏 X、補編 B、南傳 N、藏外 ZW、選集 CC、嘉興藏 J 共 **4,303 部、17,870 卷**全文（CBETA 2026.R2）；另有 591 條目錄條目導向 CBETA Online。衍生內容：AI 導讀、佛經故事 6,200+ 則、精華教授 3,900+ 條、佛學辭典 5 部。

## 直接用 URL

不透過 agent 也能用，完整文件見 https://deerpark.app/apidocs ：

```bash
curl "https://deerpark.app/api/search/title?q=心經"           # 經題搜索（支援簡體、拼音）
curl  https://deerpark.app/api/v1/work/T0251                   # 一部經的資訊與 AI 導讀
curl  https://deerpark.app/api/v1/text/T0251/1                 # 經文 Markdown 純文字
curl "https://deerpark.app/api/v1/fts/works/色即是空"           # 哪些經包含這句
curl  https://deerpark.app/api/v1/dict/lookup/般若              # 佛學辭典
```

## 使用規範

- 經文來自 [CBETA](https://cbeta.org)（CC BY-NC-SA 3.0）；deerpark 的導讀、故事、精華、專欄為 CC BY-NC-SA 4.0。請在輸出中保留出處與連結。
- 全文檢索走資料庫，請節制：一次任務數次查詢即可，不要用迴圈掃全藏。經目、導讀、目錄、經文內容都有 CDN 快取，可以放心用。
- 導讀、故事、精華是 AI 生成的現代解讀，不是經文本身；引用時請區分。

## 給 agent 的安裝指引

如果你是 AI agent，用戶要你安裝這個 skill：

1. 這個 repo 的根目錄就是 skill 資料夾（`SKILL.md` + `references/`），整個 clone 下來即可，不要只抓單一檔案。
2. 放到你自己的使用者層級 skills 目錄，資料夾命名為 `deerpark`。常見位置：Claude Code `~/.claude/skills/`、Codex `~/.codex/skills/`、Cursor `~/.cursor/skills/`、Gemini CLI `~/.gemini/skills/`、GitHub Copilot `~/.copilot/skills/`、OpenCode `~/.config/opencode/skills/`；若用戶要裝在專案內，Claude Code 用 `.claude/skills/`，其他多數用 `.agents/skills/`。
3. 不確定目錄時，改跑 `npx skills add briansun/deerpark-skill -g -y`，它會偵測你所在的 agent。
4. 裝完告訴用戶：輸入 `/deerpark` 或直接問佛經問題即可觸發；之後用 `git pull` 更新。

```bash
git clone https://github.com/briansun/deerpark-skill.git ~/.claude/skills/deerpark   # Claude Code 範例
```

## 回報問題

端點行為、skill 內容有誤，或希望增加新的資料入口，請開 [issue](https://github.com/briansun/deerpark-skill/issues)。

## License

MIT © Brian Sun
