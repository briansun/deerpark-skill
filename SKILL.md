---
name: deerpark
description: 查詢與閱讀漢文大藏經（CBETA 佛經）— 透過 https://deerpark.app 的公開 URL 搜索經題、全文檢索經文出處、讀取經文（Markdown）、查目錄與 AI 導讀、查佛學辭典、找佛經故事與精華教授。當用戶問到佛經、佛教經典、某句話出自哪部經、某部經講什麼、佛教名相解釋、譯者作品、想引用或下載佛經時使用。Use for Chinese Buddhist Canon / CBETA / Tripitaka / sutra lookup, full-text search, and reading.
---

# deerpark — 漢文大藏經 Agent Skill

deerpark.app 是一個漢文佛經（CBETA 大藏經）閱讀網站，所有資料都能用公開 URL 直接取得，
**不需要 API key、不需要登入、不需要安裝任何 CLI**。用 `curl`、`fetch` 或你手上的網頁抓取工具即可。

- 全文收錄 **4,303 部、17,862 卷**經論（大正藏 T、卍續藏 X、補編 B、南傳 N、藏外 ZW、選集 CC、嘉興藏 JA/JB），約 1.9 億字，文言文，繁體。
- 另有 **522 條**只有目錄條目（無全文）的 CBETA 作品（乾隆藏 L、印順著作 Y、高麗藏 K 等），可導向 CBETA Online。
- 衍生內容：AI 導讀、佛經故事（6,200+ 則）、精華教授（3,900+ 條）、佛學辭典（5 部）、專欄文章。
- 授權：經文 CBETA CC BY-NC-SA 3.0；本站其他文章 CC BY-NC-SA 4.0。引用時請附上 URL。

## 核心概念

| 名詞 | 說明 | 例子 |
|------|------|------|
| **id** | CBETA 經號：藏經代碼 + 編號 | `T0235`（金剛經）、`T0251`（心經）、`X0001`、`B0023` |
| **work_id** | 少數端點用帶前綴的形式 | `cbeta:T0235`（stories / highlights / search/fulltext 用） |
| **juan** | 卷。一部經 1～600 卷，每卷是一個獨立文件 | 金剛經只有 1 卷；大般若經 600 卷 |
| **lb** | CBETA 行號（冊/頁/欄/行），學術引用用 | `0748c15` = 第 748 頁 c 欄第 15 行 |

## 決策樹：先做什麼

1. **知道經名，想找經號** → `GET /api/search/title?q=金剛經`（靜態資料，最快、最便宜）
2. **想認識一部經（講什麼、誰譯、幾卷、有無導讀）** → `GET /api/v1/work/T0235`
3. **想讀經文** → `GET /api/v1/text/T0235/1`（Markdown，一卷 5K–15K 字，可直接放入上下文）
4. **一句話出自哪部經** → 名句先查 `GET /api/highlights?search=皆是虛妄`（帶出處）；有猜測就 `GET /api/v1/fts/T0235/皆是虛妄` 驗證；沒頭緒才 `GET /api/v1/fts/works/皆是虛妄?limit=100` 列候選（按命中數排，原典常在後面，挑編號小的 T 藏經律論驗證）
5. **解釋佛教名相** → `GET /api/v1/dict/suggest/如來藏` → `GET /api/v1/dict/lookup/如來藏`
6. **找故事 / 名句** → `GET /api/stories?search=布施` / `GET /api/highlights?search=空`
7. **給人類讀者的連結** → `https://deerpark.app/cbeta/T0235`（單卷）或 `https://deerpark.app/cbeta/T0262/3`（多卷第 3 卷）

## 端點速查

Base URL：`https://deerpark.app`。所有端點都是 `GET`，回 JSON（除非另註明）。URL 中的中文請 percent-encode（`curl` 會自動處理 `-G --data-urlencode`，路徑參數可直接放中文，多數 HTTP 客戶端會自動編碼）。

### 1. 經目與元數據（靜態，無資料庫成本，可放心多打）

```bash
# 經題搜索：標題 / 別名 / 譯者 / 經號，支援簡體、拼音（jgj = 金剛經）
curl -s "https://deerpark.app/api/search/title?q=金剛經&limit=10"
# → {"results":[{"id":"cbeta:T0235","title":"金剛般若波羅蜜經","alias":"金剛經","byline":"後秦 鳩摩羅什譯","sections":1,"chars":5191,"category":"般若部"},…],"total":76}
#   sections = 卷數；id 帶 cbeta: 前綴，拿去 /api/v1/* 時要去掉

# 全部經目（4,300 條、約 490 KB）。不要直接印到上下文，存檔後用 jq/grep
curl -s https://deerpark.app/api/v1/allworks -o /tmp/allworks.json
jq -r '.[] | select(.byline|test("玄奘")) | "\(.id)\t\(.title)\t\(.juans|length)卷"' /tmp/allworks.json
```

### 2. 認識一部經

```bash
curl -s https://deerpark.app/api/v1/work/T0235
```
返回：`id, title, alias, byline, canon, canonName, category, juans[], chars, creators[]{name,dynasty,url}, summary`（AI 部級導讀）、`juanSummaries[]{juan,summary}`（卷級導讀）、`urls{read,toc,text,html,search,stories,highlights,download}`。
未收錄全文的作品回 `hosted:false` 與 `urls.cbetaOnline`，不是 404。

```bash
# 目錄（章品層級 + 每卷起始行號）
curl -s https://deerpark.app/api/v1/toc/T0262
# → {"juans":[{"file":"T02nT0262","lb":"0001a03","juan":1,"title":"御製大乘妙法蓮華經序"},…],
#    "mulu":[{"indent":1,"title":"御製大乘妙法蓮華經序","juan":1,"lb":"0001a03"},{"indent":1,"title":"1 序品","juan":1,"lb":"0001c18"},…]}
```
目錄顆粒度因經而異：法華、華嚴有品目；楞嚴（T0945）等問答體的經只有序文條目；有些經 `mulu` 為空。沒有品目不代表資料截斷。

### 3. 讀經文

```bash
# Markdown 純文字（推薦給 Agent）。章品 → 標題，偈頌 → blockquote，頁尾附出處與閱讀 URL
curl -s https://deerpark.app/api/v1/text/T0235/1
curl -s https://deerpark.app/api/v1/text/T0235      # 不帶卷號 → 302 到第一卷（curl 加 -L）

# 原始 HTML（含 CBETA 行號 <span class="lb" id="T08n0235_p0748c15">，體積是 Markdown 的 15 倍）
curl -s https://deerpark.app/api/v1/html/T0235/1
```
一卷通常 5,000–15,000 字。讀多卷長經時**逐卷取、逐卷處理**，不要一次拉幾十卷。

### 4. 全文檢索（走資料庫，請節制：一次任務數次即可，勿並發轟炸）

```bash
# 跨經搜索：哪些經包含這段話（按命中數排序，最多 100 部）
curl -s "https://deerpark.app/api/v1/fts/works/一切有為法?limit=10"
# → {"found":864,"works":[{"search_results":56,"id":"T1509","title":"大智度論","byline":"…","juans":[…]},…]}

# 經內搜索：這句話在該經第幾卷、前後文是什麼（<mark> 標出命中）
curl -s "https://deerpark.app/api/v1/fts/T0235/應無所住?limit=20"
# → {"found":2,"results":[{"juan":1,"lb":"","paragraph":"…菩薩於法<mark>應無所住</mark>行於布施…"}]}

# 片段搜索（回傳命中的 chunk 與高亮，一次拿到多部經的上下文）
curl -s "https://deerpark.app/api/search/fulltext?q=應無所住而生其心&limit=10"
# → {"results":[{"workId":"cbeta:T0235","sectionNum":1,"text":"…<mark>…</mark>…","title":"金剛般若波羅蜜經","byline":"…","category":"般若部"}],"total":N,"totalCapped":bool}
```
規則：
- 自動簡→繁、自動去標點；**單字**搜索走慢路徑且可能超時（`timedOut:true`），請用 2–8 字的詞。
- 搜索詞是 BM25 詞元匹配，不是嚴格片語：用**不跨標點的短語**最準（「皆是虛妄」優於「凡所有相皆是虛妄」）。若回傳段落沒有 `<mark>`，多半是原文在詞中間有標點，chunk 仍是對的，去讀該卷 `text` 確認。
- 判斷「原出處」：`fts/works` 按命中段落數排序，命中多的常是後世註疏、語錄（X、B、JB 藏，T1993 之後的禪宗語錄）在引用；原典通常是編號較小的 T 藏經（T0001–T1692 為經律論），可能只命中一次而排在很後面。先用 `highlights?search=` 或自己的猜測，再用 `fts/{id}/{term}` 確認。
- `total` 最多計到 500（`totalCapped:true`）。

### 5. 佛學辭典（陳義孝、法相辭典、三藏法數、丁福保、佛光）

```bash
curl -s https://deerpark.app/api/v1/dict/suggest/如來藏      # 候選詞（自動簡→繁，最多 50）
curl -s https://deerpark.app/api/v1/dict/lookup/如來藏       # 精確詞條，需繁體、需完全匹配
# → {"word":"如來藏","data":[{"dict":"陳義孝佛學常見辭彙","expl":"<p>…</p>"},{"dict":"三藏法數","expl":"…"},…]}
```
`lookup` 不做簡繁轉換，先用 `suggest` 拿到正確詞形。`expl` 是 HTML 片段，去標籤即可。

### 6. 佛經故事與精華教授（AI 從經文萃取，含出處）

```bash
# 故事：search / category（本生、譬喻、因緣、果報、度化、修行、神通）/ work_id / tag / page / limit(≤100)
curl -s "https://deerpark.app/api/stories?search=布施&limit=5"
curl -s "https://deerpark.app/api/stories?work_id=cbeta:T0209&limit=20"
curl -s https://deerpark.app/api/stories/random          # 隨機 4 則，含 teaching 全文
# → {"total":885,"stories":[{"id":6344,"title_alt":"…","summary":"…","category":"因緣","tags":["慈悲","布施"],"work_id":"cbeta:N0004","section_num":5,"work_title":"長部經典"}],"categoryStats":[…],"tagStats":[…]}

# 精華：search / category（名句、教學、智慧、場景、修行、大願）/ work_id / tag / page / limit
curl -s "https://deerpark.app/api/highlights?work_id=cbeta:T0235&limit=30"
curl -s https://deerpark.app/api/highlights/random
# → {"total":28,"highlights":[{"id":7,"title":"凡所有相，皆是虛妄","original_text":"…","explanation":"…","category":"名句","tags":[…],"work_id":"cbeta:T0235","section_num":1}]}
```
人類閱讀頁：`/stories/{id}`、`/highlights/{id}`。

### 7. 下載與人類頁面

```bash
curl -sI https://deerpark.app/api/v1/download/pdf/T0945   # pdf | epub | mobi → 302 到 CBETA 官方檔案
```

| 頁面 | URL |
|------|-----|
| 閱讀（單卷 / 多卷目錄頁） | `/cbeta/{id}` |
| 閱讀多卷經第 n 卷 | `/cbeta/{id}/{n}` |
| 只有目錄條目的作品 | `/cbeta/catalog/{id}` |
| 分類瀏覽 | `/cbeta/category/般若部` |
| 譯者/作者作品列表 | `/creator/鳩摩羅什` |
| 站內搜索頁 | `/search/{term}` |
| 專欄文章（讀經筆記、專題研究、歷史人物） | `/blog`、`/blog/{slug}` |
| 讀經指南 / 關於 | `/guide`、`/about` |
| 給 LLM 的站點說明 | `/llms.txt` |

## 引用格式

回答用戶時，請給出可驗證的出處：

> 「應無所住而生其心」——《金剛般若波羅蜜經》，後秦鳩摩羅什譯（CBETA T0235）
> https://deerpark.app/cbeta/T0235

多卷經加卷號：《妙法蓮華經》卷三（CBETA T0262）https://deerpark.app/cbeta/T0262/3
需要精確到行時用 lb：T08n0235_p0749c22（冊 08、經 0235、頁 749、欄 c、行 22）。

## 使用規範

- 經文是**文言文繁體**。用戶用簡體提問時，搜索端點會自動轉換，但回覆引文請保留原文繁體。
- 先用靜態端點（title 搜索、allworks、work、toc）定位，再打全文檢索；全文檢索每次任務控制在幾次以內，不要用迴圈掃全藏。
- `allworks` 約 490 KB，`html` 一卷約 100 KB：不要直接把它們塞進上下文，落地成檔案後再抽取。
- 不要拿 deerpark 的 AI 導讀 / 故事 / 精華當作經文本身；它們是現代解讀，引用時註明「deerpark 導讀」。
- 錯誤：`404 {"error":"Work not found"}` 表示經號不存在（注意大小寫與補零：`T0235` 不是 `T235`）；`{"error":"Juan not found"}` 表示卷號不在 `juans` 內。若收到的是 **HTML 404 頁面而非 JSON**，代表路徑打錯或該端點尚未上線，不要反覆重試同一個 URL。
- 備援路徑：`/api/v1/work` 不可用時，用 `/api/search/title` 拿標題、譯者、分類、卷數；`/api/v1/text` 不可用時，用 `/api/v1/html` 再剝標籤：
  `curl -s .../api/v1/html/T0235/1 | sed -e 's/<span class="lb"[^>]*>[^<]*<\/span>//g' -e 's/<\/p>/\n\n/g' -e 's/<[^>]*>//g'`
- 朝代異名：同一政權在不同端點可能寫法不同（「姚秦」=「後秦」、「元魏」=「北魏」、「劉宋」=「宋」），不是資料衝突。
- 更完整的回應欄位與範例見 [references/endpoints.md](references/endpoints.md)；藏經代碼與收錄範圍見 [references/canons.md](references/canons.md)；常見任務的完整流程見 [references/recipes.md](references/recipes.md)。
