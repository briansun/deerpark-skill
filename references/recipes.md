# 常見任務範例

每個範例都只用 `curl` + `jq`（或任何能發 GET 請求的工具）。URL 中的中文可以直接放在路徑裡，`curl` 會自動編碼；query string 用 `--data-urlencode` 最保險。

## 1. 「這句話出自哪部經？」

用戶問：「凡所有相皆是虛妄」是哪部經說的？三條路，由快到慢：

```bash
# 路徑 A：名句多半已被收進「精華教授」，直接帶出處（最快，適合著名句子）
curl -s -G "https://deerpark.app/api/highlights" --data-urlencode "search=皆是虛妄" --data-urlencode "limit=5" \
  | jq -r '.highlights[] | "\(.work_id) 卷\(.section_num)\t\(.work_title)\t\(.original_text)"'
# cbeta:T0235 卷1  金剛般若波羅蜜經  凡所有相，皆是虛妄；若見諸相非相，則見如來。
# cbeta:T1509 卷32 大智度論          …皆是虛妄。以是故，佛說：心力為大…

# 路徑 B：你已有猜測（例如知道這是金剛經），直接在該經內驗證
curl -s "https://deerpark.app/api/v1/fts/T0235/皆是虛妄?limit=3" | jq -r '.results[] | "卷\(.juan)：\(.paragraph)"'
# 卷1：…佛告須菩提：「凡所有相，<mark>皆是虛妄</mark>；若見諸相非相，則見如來。」…

# 路徑 C：毫無頭緒。列出所有包含這句的經，再從中挑原典驗證
curl -s "https://deerpark.app/api/v1/fts/works/皆是虛妄?limit=100" \
  | jq -r '.works[] | "\(.id)\t\(.search_results)\t\(.title)"'
# B0023  25  金剛般若波羅蜜經講義   ← 近代講義，引用最多
# X1571  10  五燈全書               ← 禪宗史書
# T1509  11  大智度論               ← 論典引用
# …（結果按命中段落數排序，原典若只出現一次會排在很後面甚至不在前 100）
```

判斷原出處的經驗法則：
- **命中次數多 ≠ 出處**。註疏（X、B、JB 藏）與禪宗語錄（T1993–T2025 一帶）大量引用經文；原典通常在 T0001–T1692（經、律、論）且編號較小。路徑 C 的清單要挑編號小的 T 藏經律論逐一用路徑 B 驗證。
- 搜索詞用**不跨標點的 2–6 字子句**。原文是「凡所有相，皆是虛妄」，搜「皆是虛妄」或「凡所有相」比搜整句準；用戶給的句子常是後人改寫（「一切有為法如夢幻泡影」原文是「一切有為法，如夢、幻、泡、影」）。
- 回傳段落若沒有 `<mark>`，通常是詞中間有標點導致高亮失敗，chunk 本身仍是命中的；讀該卷 `text` 確認。
- 找到候選後，讀該卷 `text` 端點確認完整原句，再引用。

回答格式：
> 出自《金剛般若波羅蜜經》（後秦鳩摩羅什譯，CBETA T0235）：「凡所有相，皆是虛妄；若見諸相非相，則見如來。」
> https://deerpark.app/cbeta/T0235
> 《大智度論》《六祖壇經》《宗鏡錄》等後世著作多有引用。

## 2. 「《金剛經》講什麼？」

```bash
# 找經號（別名 → 正式經題）
curl -s -G "https://deerpark.app/api/search/title" --data-urlencode "q=金剛經" --data-urlencode "limit=5" \
  | jq -r '.results[] | "\(.id)\t\(.title)\t\(.byline)\t\(.sections)卷"'
# cbeta:T0235  金剛般若波羅蜜經  後秦 鳩摩羅什譯  1卷   ← 通常用戶指的是這部
# cbeta:T0236a 金剛般若波羅蜜經  元魏 菩提流支譯 …    ← 同經異譯

# 拿導讀與基本資料
curl -s "https://deerpark.app/api/v1/work/T0235" | jq '{title, byline, category, chars, juans, summary}'

# 精華教授（經中名句 + 解讀），適合快速掌握重點
curl -s "https://deerpark.app/api/highlights?work_id=cbeta:T0235&limit=30" | jq -r '.highlights[] | "- \(.original_text)"'
```
只有 5,191 字，直接讀全文也行：`curl -s https://deerpark.app/api/v1/text/T0235/1`。

## 3. 讀一部多卷的經

用戶：幫我看看《妙法蓮華經》的〈觀世音菩薩普門品〉。

```bash
# 目錄：找到品在第幾卷
curl -s "https://deerpark.app/api/v1/toc/T0262" | jq -r '.mulu[] | "\(.juan)\t\(.title)"' | grep 普門
# 7  25 觀世音菩薩普門品

# 只讀那一卷（約 1 萬字）
curl -s "https://deerpark.app/api/v1/text/T0262/7" > /tmp/T0262-7.md
# 在檔案裡定位「普門品」開始的位置再讀，不必把整卷都放進上下文
```

要通讀整部 7 卷的經：逐卷 `text/T0262/1` … `/7`，每讀一卷做一次摘要再讀下一卷。大般若經 600 卷、大智度論 100 卷這類巨著，先讀 `/api/v1/work` 的導讀與 `toc`，再挑卷。

## 4. 解釋一個佛教名相

用戶：什麼是「阿賴耶識」？

```bash
# 先看辭典有哪些相關詞條
curl -s "https://deerpark.app/api/v1/dict/suggest/阿賴耶識" | jq -r '.[]' | head
# 精確查（用 suggest 回來的繁體詞形）
curl -s "https://deerpark.app/api/v1/dict/lookup/阿賴耶識" | jq -r '.data[] | "【\(.dict)】\(.expl | gsub("<[^>]*>"; ""))"'

# 想看經論原文怎麼用這個詞：
curl -s "https://deerpark.app/api/v1/fts/works/阿賴耶識?limit=10" | jq -r '.works[] | "\(.id)\t\(.title)\t\(.search_results)"'
# T1579 瑜伽師地論、T1585 成唯識論、T1594 攝大乘論 …
```

## 5. 找適合講給孩子聽的佛經故事

```bash
curl -s -G "https://deerpark.app/api/stories" --data-urlencode "category=本生" --data-urlencode "search=兔" --data-urlencode "limit=5" \
  | jq -r '.stories[] | "\(.id)\t\(.title_alt)\t\(.work_title) 卷\(.section_num)\n  \(.summary)"'
# 每則故事都有 work_id / section_num，可以用 text 端點回到原文：
curl -s "https://deerpark.app/api/v1/text/T0152/3" | grep -n "兔" | head
```
人類可讀頁面：`https://deerpark.app/stories/{id}`。

## 6. 某位譯者譯了哪些經

```bash
# 譯者名在 byline 裡，用經題搜索即可（支援簡體、部分匹配）
curl -s -G "https://deerpark.app/api/search/title" --data-urlencode "q=玄奘" --data-urlencode "limit=100" \
  | jq -r '.results[] | "\(.id)\t\(.title)\t\(.sections)卷\t\(.category)"' | sort

# 或從 allworks 本地過濾（一次下載，之後零成本）
curl -s https://deerpark.app/api/v1/allworks -o /tmp/allworks.json
jq -r '.[] | select(.byline | test("玄奘")) | "\(.id)\t\(.title)"' /tmp/allworks.json | wc -l
```
人類頁面：`https://deerpark.app/creator/玄奘`（含朝代與全部作品）。

## 7. 主題研究：某個概念在不同經典中的說法

用戶：比較幾部經對「無我」的說法。

```bash
# 1. 哪些經談得最多（命中段落數）
curl -s "https://deerpark.app/api/v1/fts/works/無我?limit=20" | jq -r '.works[] | "\(.search_results)\t\(.id)\t\(.title)"'

# 2. 挑代表性的經（阿含 / 般若 / 唯識各一），各取幾段上下文
for id in T0099 T0235 T1585; do
  echo "== $id"; curl -s "https://deerpark.app/api/v1/fts/$id/無我?limit=3" | jq -r '.results[] | "卷\(.juan)：\(.paragraph)"'
done
```
引用時每段都附 `https://deerpark.app/cbeta/{id}/{juan}`。

## 8. 產出可核對的引用

原則：**經題 + 譯者 + CBETA 經號 + 卷 + URL**。需要到行時，用 HTML 版找 `lb`：

```bash
# 把 HTML 拆成「行號 / 關鍵詞」序列，關鍵詞前一行就是它所在的 CBETA 行號
curl -s "https://deerpark.app/api/v1/html/T0235/1" \
  | grep -o 'class="lb" id="[^"]*"\|應無所住' | grep -B1 '應無所住' | head -2
# class="lb" id="T08n0235_p0749a12"
# 應無所住
```
寫成：《金剛般若波羅蜜經》，CBETA T08, no. 235, p. 749a12。

## 9. 下載檔案給用戶

```bash
curl -s -o /dev/null -w '%{redirect_url}\n' "https://deerpark.app/api/v1/download/epub/T0262"
# https://cbdata.dila.edu.tw/stable/download/epub/T/T0262.epub
```
直接把這個 CBETA 官方連結給用戶即可（PDF 適合列印、ePub 適合 Apple Books、MOBI 適合 Kindle）。

## 10. 用戶用簡體提問

所有搜索端點都會自動簡→繁（`金刚经` → `金剛經`；`应无所住` → `應無所住`），唯一例外是 `dict/lookup`（先過 `dict/suggest`）。
回覆時引文保持繁體原文，解說可用用戶的語言。
