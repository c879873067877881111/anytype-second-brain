# /capture flow

## Input

`$ARGUMENTS` = 自然語言文字，可能含多個概念。

## Step 1 — Extract candidate topics

從輸入抽 1–N 個 candidates。每個帶：

- `name`: 標準化大小寫的概念名（例：`PyTorch` 不是 `pytorch`）
- `aliases[]`: 原文出現的所有寫法（含 `name` 自己的變體）
- `summary`: 對該概念說了什麼，**一句話**
- `body_addition`: 該概念對應的完整內容塊（merge 用）

抽取原則：
- 名詞短語優先（`autograd`、`dynamic graph`），不要動詞、不要句子
- 同一句話可能對應多個 candidate
- 過於泛化的詞（`code`、`概念`、`東西`）丟掉

## Step 2 — Match each candidate

對每個 candidate，**兩條 search 都要做**（search-space 不查 aliases）：

a. **Name search**：`mcp__anytype__API-search-space`
   - `space_id`: 見 SKILL.md
   - `query`: candidate.name
   - `types`: `["topic"]`
   - `limit`: 10

b. **Alias scan**：`mcp__anytype__API-search-space`
   - `query`: ""（空字串）
   - `types`: `["topic"]`
   - `limit`: 200
   - sort: 預設 last_modified desc
   - client-side 過濾：取每個結果的 `aliases` 字串，split by `, ` 後 trim，檢查是否含 candidate.name 或任一 candidate.aliases（case-insensitive）

c. 合併 a + b 結果，按 object id 去重。

d. **LLM 判斷**是否同主題：
   - name 完全相同（case-insensitive） → 同
   - 候選名 ∈ 結果的 aliases（已過濾） → 同
   - 結果的 `description` + body snippet 與 candidate.summary 描述同一概念 → 同
   - 否則 → 不同

e. 結果：選一個 match（最高信心），或 `no_match`。

## Step 3a — No match → create

`mcp__anytype__API-create-object`：

```
space_id: <SKILL.md>
type_key: topic
name: candidate.name
body: candidate.body_addition          ← 頂層 body 欄位 = native markdown body
properties:
  - {key: description, text: candidate.summary}
  - {key: status, select: "<seedling tag id from SKILL.md>"}
  - {key: aliases, text: ", ".join(candidate.aliases)}
  - {key: last_merged_at, date: "<today ISO 8601, e.g. 2026-05-07T00:00:00Z>"}
```

**注意**：
- `select` 要傳 **tag ID**（SKILL.md 有完整對照表），不是 name
- description 會被 Anytype prepend 到 markdown body 顯示，所以要寫得對讀者也有意義

記下回傳的 object id，後面連 Daily 用。

## Step 3b — Match found → merge

i. `mcp__anytype__API-get-object` 拿現有 Topic。從回傳取出：
   - `markdown` 欄位（已含 description prepend + 真正 body）
   - 各 property: `aliases`, `status` (select.key), `description`

   **取真正 body**：把 markdown 開頭的 description 行剝掉，剩下的才是要 merge 的 body。

ii. **Merge body**（最重要，認真做）：

   - **不要** append `## 2026-05-07` 章節
   - **不要** 加 `Update:` 或 `New:` 前綴
   - 把 `body_addition` 的訊息**融入**現有段落
   - 新事實 → 補進對應主題的段落
   - 既有句子模糊但新資料更精確 → 改寫舊句
   - 與既有內容矛盾 → 兩個說法**並列**，標註出處（如有 Source）；別覆蓋

   原則：merge 後，沒看過該 Topic 的人讀起來像一篇統整文章。

iii. **Aliases**: 既有 aliases ∪ candidate.aliases（去重，保留原大小寫，逗號 + 空格 join）。

iv. **Status**:
   - `seedling` → `budding`
   - `budding` → `budding`（不動）
   - `evergreen` → `evergreen`（不動）

v. **Description**: 若舊 description 過時（與 merge 後 body 重點不符），更新；否則不動。

vi. `mcp__anytype__API-update-object`：

```
object_id: <existing topic id>
markdown: <merged body>                ← 整個覆寫，merge 必須在 LLM 端先做完
properties:
  - {key: aliases, text: <merged aliases>}
  - {key: status, select: "<status tag id>"}
  - {key: last_merged_at, date: "<today ISO 8601>"}
  - (description 如有更新)
```

**關鍵**：`markdown` 欄位會**整段覆寫** body，不是 append。所以一定要先 get 拿原內容、merge 完再寫回。

## Step 4 — Link to today's Daily

對所有 touched topic ids（步驟 3 處理過的）：

a. today = YYYY-MM-DD（從系統時間，不要從 input 推測）

b. `search-space` query=today, types=`["daily"]`, limit=3。Daily name 應該 = today。

c. 找到 → `get-object` 拿 `touched_on` 現值 → ∪ touched topic ids（去重） → `update-object` 用整個 array 替換。

d. 找不到 → `create-object` type=daily, name=today, properties: entry_date (ISO 8601), touched_on=touched topic ids。

## Step 5 — Auto-link related Topics

如果一次 capture 產出多個 touched topics（≥2），它們之間通常相關。

對 touched topics 兩兩配對：對每個 topic，把其他 touched topic id 加入它的 `relates_to`（先 get 拿原 array 再 union）。

**注意**：`relates_to` update 是覆寫整個 array，要保留舊值。

**不要**對既有 Topic 做猜測式配對（那是 `/connect` 的工作）；只在「同一次 capture 內」連結。

> 雖然 backlinks 自動雙向，但 relates_to 是語意明確的同類關聯，仍要雙向 set 才能在 UI 的 relates_to section 看到反向。

## Step 6 — Report

簡短輸出，每行一個 topic：

```
+ <name>           (新建, seedling)
~ <name>           (merged, seedling → budding)
~ <name>           (merged, budding)
+ aliases: a, b    (若有新 alias)
↔ <topic1> ↔ <topic2>   (若同次 capture 有 ≥2 topic)
→ Daily 2026-05-07 (touched N topics)
```

不要長篇大論，使用者能看出「這次寫進去什麼」就好。

## Anti-patterns

- ❌ append `## 2026-05-07` 章節到 body
- ❌ 為每個 capture 建 atomic note
- ❌ select 傳 tag name 而非 tag ID（會建立失敗或忽略）
- ❌ create 時用 `properties: [{key: body, text: ...}]`（沒有 body property，要用頂層 `body`）
- ❌ update 時用 `body` 欄位（沒這個欄位，要用 `markdown`）
- ❌ update markdown 時忘記先 get 舊內容（會整個覆蓋）
- ❌ 跳過 alias scan（search 不查 aliases，會誤判沒匹配而建重複頁）
- ❌ 矛盾資料時靜默覆蓋舊內容（要並列）
- ❌ 對遠房親戚 Topic 自動加 `relates_to`（步驟 5 只連同一次 capture 內的）
- ❌ 把整段 input 當 description（description 是一句話 summary）
