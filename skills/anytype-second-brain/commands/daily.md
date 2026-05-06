# /daily flow

## Input

`$ARGUMENTS` 可選：
- **空** → 顯示今日 Daily 內容
- **有文字** → append 到今日 Daily 的 body（時間順序日誌）

## Step 1 — Find today's Daily

```
today = YYYY-MM-DD（系統時間）
now = HH:MM（24h）
```

`mcp__anytype__API-search-space`:
- `query`: today
- `types`: `["daily"]`
- `limit`: 3

從結果找 **`name` == today** 的 Daily（不要 partial match — 「2026-05」會撈到一堆）。

## Step 2a — No Daily today

- **`$ARGUMENTS` 空** → 輸出：
  ```
  📅 today 還沒記錄。
  /capture <想法> 或 /daily <文字> 開始今天。
  ```
  結束。**不要建空 Daily**（會留空殼）。

- **`$ARGUMENTS` 非空** → `mcp__anytype__API-create-object`:
  ```
  type_key: daily
  name: <today YYYY-MM-DD>
  body: "[<HH:MM>] <$ARGUMENTS>"
  properties:
    - {key: entry_date, date: "<today ISO 8601>"}
  ```
  跳到 Step 3 顯示。

## Step 2b — Daily exists

i. `mcp__anytype__API-get-object` 拿完整資料。

ii. **取出真正 body**：
   - 從 properties 拿 `description` 字串（可能空字串）
   - 從回傳 `markdown` 欄位剝掉 description prepend
     - 若 description 非空：去掉 markdown 開頭的 `<description>\n` 或 `<description>   \n`（Anytype 用 `\n` 結尾，可能含尾隨空白）
     - 若 description 空：markdown 開頭直接是 body
   - 把結尾的尾隨空白/換行 trim 掉
   - 結果就是 `current_body`

iii. **`$ARGUMENTS` 空** → 跳到 Step 3 顯示，不寫入。

iv. **`$ARGUMENTS` 非空**：
   - 組新 body：
     - 若 `current_body` trim 後為空（Daily 由 capture 自動建但沒寫過 body 的常見 case）：
       `new_body = "[<HH:MM>] <$ARGUMENTS>"`
     - 否則：
       `new_body = current_body.rstrip() + "\n[<HH:MM>] <$ARGUMENTS>"`
   - **用單 `\n` 不要 `\n\n`**：測試發現 Anytype 會把空白行壓掉、每行尾自動加 `<br>` (hard break)，效果跟 `\n\n` 一樣但 source 乾淨。
   - `mcp__anytype__API-update-object`:
     ```
     object_id: <daily id>
     markdown: <new_body>
     ```
   - **不更新** `touched_on`（這是 capture/ingest 的事，/daily 只寫文字）
   - **不更新** `description`（保持長期穩定）

## Step 3 — Display

`<body>` 用 **Step 2b ii 的 `current_body`**（已剝掉 description prepend）。Step 2a 走來的（剛 create）用本次寫入的 body。**不要直接顯示 `markdown` 欄位**，否則 description 會被印兩次（一次在標題行的 `· <description>`、一次在 body 開頭）。

```
📅 today<description if non-empty: ` · <description>`>

<body>

<count> entries · <touched_count> topics touched
```

- `count` = body 中 `[HH:MM]` 行首出現次數。pattern `\[\d{2}:\d{2}\]` 全文 scan；若用 `^\[\d{2}:\d{2}\]` 必須配 multiline flag (`(?m)`)，否則只 match 字串開頭 1 次。
- `touched_count` = `touched_on` array 長度（不要 enumerate Topic names — Anytype UI 自己顯示）

## Anti-patterns

- ❌ 無 args 也建空 Daily（會留空殼污染 space）
- ❌ append 不帶 `[HH:MM]` 時間標記（事後讀不出時序）
- ❌ 用 `\n\n` 隔開兩筆（Anytype 會吞空行 — 用單 `\n` 即可，UI 自動 hard break）
- ❌ replace 整個 body（Daily 是時間軸，必須保留歷史 — 跟 Topic 的 merge-then-rewrite 規則相反）
- ❌ 寫進 description（description 是 page subtitle，長期穩定；日誌寫 body）
- ❌ 自動加 `touched_on`（那是 capture/ingest 的職責；/daily 不該動 Topic 關聯）
- ❌ partial match Daily name（query="2026-05" 會撈整個月）
- ❌ 用過去日期 — `/daily` 只操作 today，要改別天請用 `/daily-edit <date>` 或直接 Anytype UI（v1 不支援）

## Examples

### A. 空 Daily，純查詢
```
/daily
```
→ search 找不到 today → 輸出「📅 2026-05-07 還沒記錄。/capture ... 或 /daily ... 開始今天。」

### B. 第一筆寫入
```
/daily 早上跟 chris 同步 GTM 計畫，我們對第三方整合有分歧
```
→ search 找不到 → create Daily, body="[09:23] 早上跟 chris 同步 GTM 計畫，我們對第三方整合有分歧"
→ Display

### C. Daily 已有內容（例如 capture 過幾個 Topic 而自動建了 Daily），補一筆 free-form
```
/daily 想到 PyTorch autograd 的 dynamic graph 設計可能受 Theano 啟發
```
→ search 找到 today's Daily（已被 capture 建過、touched_on 有 3 個 Topic）
→ get markdown = "" (沒 description, capture 沒寫 body)
→ new_body = "[14:45] 想到 PyTorch autograd 的 dynamic graph 設計可能受 Theano 啟發"
→ update markdown
→ Display: "1 entries · 3 topics touched"

### D. 第二筆 append
```
/daily chris 回我的 thread，他主要擔心 vendor lock-in
```
→ search 找到 → get current_body = "[09:23] 早上跟 chris 同步..."
→ new_body = current_body + "\n[16:02] chris 回我的 thread，他主要擔心 vendor lock-in"
→ update
→ Display: "2 entries · 0 topics touched"
