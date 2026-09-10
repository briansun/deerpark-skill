# deerpark.app 端點參考

所有端點：`GET`、無需認證、回 `application/json`（除非另註明）。未設定 CORS 標頭，請從伺服器端 / 命令列呼叫，不要在瀏覽器頁面裡跨域 fetch。
Base URL：`https://deerpark.app`。線上文件：https://deerpark.app/apidocs

錯誤回應統一為 `{"error": "..."}`，配合 4xx / 5xx 狀態碼。

## 經目與元數據（靜態 JSON，無資料庫成本）

### `GET /api/search/title?q={term}&limit={n}`

搜索經題、別名、備用標題、譯者（byline）、經號。支援：
- 簡體 → 自動轉繁體
- 拼音：完整或首字母（`jingangjing`、`jgj` 都能匹配「金剛經」）
- 排序：完全匹配 > 漢字直接匹配 > 拼音匹配 > 標題開頭匹配 > 字數少者優先

```json
{
  "results": [
    {
      "id": "cbeta:T0235",
      "title": "金剛般若波羅蜜經",
      "alias": "金剛經",
      "byline": "後秦 鳩摩羅什譯",
      "sections": 1,
      "chars": 5191,
      "category": "般若部"
    }
  ],
  "total": 76,
  "query": "金剛經"
}
```
注意 `id` 帶 `cbeta:` 前綴；其他 `/api/v1/*` 端點用不帶前綴的 `T0235`。`limit` 預設 20。

### `GET /api/v1/allworks`

所有收錄全文的經（4,303 條，約 490 KB）。適合一次下載後本地過濾。

```json
[
  {
    "id": "T0001",
    "title": "長阿含經",
    "byline": "後秦 佛陀耶舍共竺佛念譯",
    "juans": [1, 2, 3, "...", 22],
    "chars": 198436,
    "alias": "..."        // 可選
  }
]
```
不含 `category`；要分類請用 `/api/search/title` 或 `/api/v1/work/:id`。

### `GET /api/v1/work/{id}`

一部經的完整資訊。

```json
{
  "id": "T0235",
  "hosted": true,
  "title": "金剛般若波羅蜜經",
  "alias": "金剛經",
  "titleAlt": null,
  "byline": "後秦 鳩摩羅什譯",
  "canon": "T",
  "canonName": "大正新脩大藏經",
  "category": "般若部",
  "juans": [1],
  "chars": 5191,
  "creators": [
    { "id": "A001583", "name": "鳩摩羅什", "dynasty": "姚秦", "workCount": 54, "url": "https://deerpark.app/creator/%E9%B3%A9%E6%91%A9%E7%BE%85%E4%BB%80" }
  ],
  "summary": "這部經記錄了佛在舍衛國祇樹給孤獨園與弟子須菩提的一場對話……",
  "juanSummaries": [ { "juan": 1, "summary": "..." } ],
  "urls": {
    "read": "https://deerpark.app/cbeta/T0235",
    "readJuan": "https://deerpark.app/cbeta/T0235",       // 多卷經為 "https://deerpark.app/cbeta/{id}/{juan}" 模板
    "toc": "https://deerpark.app/api/v1/toc/T0235",
    "text": "https://deerpark.app/api/v1/text/T0235/1",
    "html": "https://deerpark.app/api/v1/html/T0235/1",
    "search": "https://deerpark.app/api/v1/fts/T0235/{term}",
    "stories": "https://deerpark.app/api/stories?work_id=cbeta:T0235",
    "highlights": "https://deerpark.app/api/highlights?work_id=cbeta:T0235",
    "download": { "pdf": "...", "epub": "...", "mobi": "..." }
  }
}
```
- `summary` / `juanSummaries` 是 deerpark 的 AI 導讀，可能為 `null` / `[]`。
- 未收錄全文的作品（乾隆藏 L、印順著作 Y 等）回 `200` 且 `hosted: false`：

```json
{
  "id": "Y0001",
  "hosted": false,
  "title": "般若經講記",
  "byline": "民國 釋印順著",
  "canon": "Y",
  "canonName": "印順法師佛學著作集",
  "category": "新編部",
  "juanCount": 3,
  "chars": 72966,
  "urls": {
    "catalog": "https://deerpark.app/cbeta/catalog/Y0001",
    "cbetaOnline": "https://cbetaonline.dila.edu.tw/zh/Y0001"
  }
}
```

### `GET /api/v1/toc/{id}`

```json
{
  "juans": [
    { "file": "T02nT0262", "lb": "0001a03", "juan": 1, "title": "御製大乘妙法蓮華經序" },
    { "file": "T02nT0262", "lb": "0010b28", "juan": 2, "title": "3 譬喻品" }
  ],
  "mulu": [
    { "indent": 1, "title": "御製大乘妙法蓮華經序", "juan": 1, "lb": "0001a03" },
    { "indent": 1, "title": "1 序品", "juan": 1, "lb": "0001c18" },
    { "indent": 1, "title": "2 方便品", "juan": 1, "lb": "0005b24" }
  ]
}
```
- `mulu` 是展平後的章品目錄，`indent` 為層級（1 起）；`juans` 每卷取該卷第一個目錄條目。品名前的數字是品序（「25 觀世音菩薩普門品」）。
- 沒有目錄資料的經，`mulu` 為 `[]`，`juans` 只有卷號。
- `file` 欄位格式目前不可靠（應為 `T09n0262`），請勿依賴；行號請用 `lb`。

## 經文內容

### `GET /api/v1/text/{id}/{juan}` → `text/markdown; charset=utf-8`

Markdown 純文字。結構：
- `# 經題` → `*譯者*` → `**卷題**`
- 章品標題（CBETA `head`）→ `##`～`####`
- 偈頌 → 連續的 `> ` 行，句與句之間以全形空格分隔
- 頁尾 `---` 之後為出處與閱讀 URL
- 已移除：行號、標點 span、校勘註腳錨點、圖片

`GET /api/v1/text/{id}`（不帶卷號）→ `302` 到第一卷。

### `GET /api/v1/html/{id}/{juan}` → `text/html; charset=utf-8`

原始 HTML（`<article class="sutra-content">`）。需要 CBETA 行號時用這個：

```html
<span class="lb" id="T08n0235_p0748c15">T08n0235_p0748c15</span>
<span class="t" l="0748c15" w="1">如是我聞<span class="pc">：</span></span>
```
- `T08n0235_p0748c15`：大正藏第 08 冊、經號 0235、頁 0748、欄 c、行 15
- `<p class="head" data-head-level="1">` 章品標題；`<div class="lg ... verse">` 偈頌
- 一卷 HTML 約為 Markdown 的 15 倍體積

### `GET /api/v1/download/{pdf|epub|mobi}/{id}` → `302`

轉向 CBETA 官方檔案，如 `https://cbdata.dila.edu.tw/stable/download/pdf/T/T0235.pdf`。

## 全文檢索（走資料庫）

三個端點都會：簡體 → 繁體、去除標點、BM25 檢索（單字改用子串匹配，較慢）。

### `GET /api/v1/fts/works/{term}?limit={≤100}&page={n}`

哪些經包含這個詞，按命中段落數排序。

```json
{
  "found": 864,
  "works": [
    { "search_results": 56, "id": "T1509", "title": "大智度論", "byline": "龍樹菩薩造 後秦 鳩摩羅什譯", "juans": [1, 2, "..."], "chars": 956053 }
  ]
}
```
`found` 是命中段落總數（不是經數）。

### `GET /api/v1/fts/{id}/{term}?limit={≤100}&page={n}`

在一部經內搜索，按出現順序排。

```json
{
  "found": 2,
  "results": [
    { "juan": 1, "lb": "", "paragraph": "…菩薩於法<mark>應無所住</mark>行於布施…" }
  ]
}
```
`lb` 目前固定為空字串；需要行號請拿 `juan` 去讀 HTML 版再定位。

### `GET /api/search/fulltext?q={term}&limit={≤100}&offset={n}`

跨經排名式片段搜索：BM25 分數 × 藏經權重（大正藏 > 卍續藏 > 其他；編號小者優先）。

```json
{
  "results": [
    {
      "id": "186384",
      "workId": "cbeta:T0235",
      "sectionNum": 1,
      "chunkIndex": 7,
      "text": "…<mark>應無所住而生其心</mark>…",
      "rawText": "…",
      "score": 12.3,
      "title": "金剛般若波羅蜜經",
      "byline": "後秦 鳩摩羅什譯",
      "category": "般若部",
      "sections": 1
    }
  ],
  "total": 37,
  "totalCapped": false,
  "query": "應無所住而生其心",
  "limit": 10,
  "offset": 0
}
```
- 候選池 500：`total` 最大 500，超過時 `totalCapped: true`
- 查詢超時回 `200` 且 `timedOut: true, results: []`，換個更具體的詞重試一次即可
- `text` 是一個 chunk（約數百字）的高亮版本，`rawText` 無標記

## 佛學辭典

### `GET /api/v1/dict/suggest/{term}` → `string[]`

前綴/包含匹配的候選詞，最多 50，簡體自動轉繁體。

### `GET /api/v1/dict/lookup/{term}`

精確匹配（區分繁簡、不做轉換）。找不到回 `404 {"error":"Word not found"}`。

```json
{
  "word": "如來藏",
  "data": [
    { "dictid": 0, "dict": "陳義孝佛學常見辭彙", "expl": "<p>真如在煩惱中……</p>" },
    { "dictid": 2, "dict": "三藏法數", "expl": "<p>藏即含藏也……</p>" },
    { "dictid": 3, "dict": "丁福保佛學大辭典", "expl": "..." },
    { "dictid": 4, "dict": "佛光大辭典", "expl": "..." }
  ]
}
```
五部辭典：陳義孝佛學常見辭彙、法相辭典、三藏法數、丁福保佛學大辭典、佛光大辭典。
加 `?html` 回 HTML 片段。

## 佛經故事與精華教授

### `GET /api/stories`

| 參數 | 說明 |
|------|------|
| `search` | 模糊匹配標題 / 摘要 / 寓意 |
| `category` | 本生、譬喻、因緣、果報、度化、修行、神通、其他 |
| `work_id` | `cbeta:T0209` |
| `tag` | 單一標籤，如 `布施` |
| `page` / `limit` | 預設 1 / 24，`limit` ≤ 100 |

```json
{
  "total": 6256,
  "stories": [
    {
      "id": 6344,
      "title_alt": "國王的慈悲祭祀",
      "summary": "摩訶偉質多王想舉行盛大的祭祀……",   // 截斷至 200 字
      "category": "因緣",
      "tags": ["慈悲", "布施", "因果"],
      "work_id": "cbeta:N0004",
      "section_num": 5,
      "work_title": "長部經典"
    }
  ],
  "categoryStats": [ { "category": "度化", "count": 1404 } ],
  "workStats":     [ { "workId": "cbeta:T2122", "title": "法苑珠林", "count": 1108 } ],
  "tagStats":      [ { "tag": "因果", "count": 1751 } ],
  "pagination": { "page": 1, "limit": 24, "total": 6256, "totalPages": 261 }
}
```
列表的 `summary` 截斷至 200 字；完整內容（含 `teaching` 寓意、`characters`）看人類頁面 `/stories/{id}`，或用 `/api/stories/random`（隨機 4 則、完整欄位）。

### `GET /api/highlights`

參數同上；`category` 為：名句、教學、智慧、場景、修行、大願。

```json
{
  "total": 3937,
  "highlights": [
    {
      "id": 7,
      "title": "凡所有相，皆是虛妄",
      "original_text": "凡所有相，皆是虛妄；若見諸相非相，則見如來。",
      "explanation": "這是《金剛經》最著名的開示之一……",
      "category": "名句",
      "tags": ["空性", "無相", "虛妄"],
      "work_id": "cbeta:T0235",
      "section_num": 1,
      "work_title": "金剛般若波羅蜜經"
    }
  ],
  "categoryStats": [], "workStats": [], "tagStats": [], "pagination": {}
}
```
`original_text` 是經文原句（可直接引用），`explanation` 是 AI 解讀；列表中兩者都截斷至 200 字，完整版（含 `significance`）看 `/highlights/{id}` 頁面或 `/api/highlights/random`。

### `GET /api/stories/random`、`GET /api/highlights/random`

各回隨機 4 則的**陣列**（不是物件），含完整欄位：
- story：`id, work_id, section_num, title, title_alt, summary, teaching, start_marker, category, tags, characters, work_title`
- highlight：`id, work_id, section_num, category, title, original_text, explanation, significance, tags, start_marker, work_title`

## 人類頁面（HTML，含 JSON-LD 結構化資料）

| 頁面 | URL |
|------|-----|
| 首頁 | `/` |
| 全部經目（可按分類篩選） | `/cbeta` |
| 分類 | `/cbeta/category/{分類名}`，如 `/cbeta/category/般若部` |
| 閱讀：單卷經 / 多卷經目錄頁 | `/cbeta/{id}` |
| 閱讀：多卷經第 n 卷 | `/cbeta/{id}/{n}` |
| 目錄條目（未收錄全文） | `/cbeta/catalog/{id}` |
| 譯者 / 作者 | `/creator/{名字}`，如 `/creator/鳩摩羅什` |
| 站內搜索結果頁 | `/search/{term}` |
| 佛經故事 | `/stories`、`/stories/{id}` |
| 精華教授 | `/highlights`、`/highlights/{id}` |
| 專欄 | `/blog`、`/blog/{slug}` |
| 讀經指南 | `/guide` |
| 關於 | `/about` |
| API 文件 | `/apidocs` |
| LLM 站點說明 | `/llms.txt` |
| Sitemap | `/sitemap/0.xml`（靜態頁）、`/sitemap/1.xml`（經文頁）… |

分類（按經律論順序）：阿含部、本緣部、般若部、法華部、華嚴部、寶積部、涅槃部、大集部、經集部、密教部、律部、毘曇部、中觀部、瑜伽部、論集部、禪宗部、淨土宗部、史傳部、事彙部、敦煌寫本部、南傳大藏經部、新編部。

## 已下線的端點

`/podcasts/*`、`/reading-list/*` 回 `410 Gone`。`/kuma.today/*`（新譯佛經）已遷移到 https://buddha.now 。
