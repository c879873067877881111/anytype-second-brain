---
name: anytype-second-brain
description: LLM-rewritten wiki in Anytype. Every input merges into existing Topic pages instead of appending new notes. Use when capturing thoughts, ingesting sources, or maintaining a living knowledge base in the "Second Brain" Anytype space. Triggered by /capture, /find, /daily, /ingest, /connect, /reconcile, /review.
---

# Anytype Second Brain

每筆新資訊應 **merge 進既有 Topic page**，不 append 新頁。建新只發生在沒有匹配時。

## Resources

| Item | ID |
|---|---|
| Space | `bafyreigbldcnynkvkjnwsplds2kj73765vtl6rmtdvj3bi7fwartlquaoq.1ozas83y3t9p0` |
| Type: Topic | `bafyreickevg3qkejtk7eedrsiidtdswqdh3fyiheuogz5ja2lo4lrw23pq` (key: `topic`) |
| Type: Source | `bafyreihtamvd3os7u3nfqzlzhwcbfn3jh2x724e6cmdebwrkqib4qic3tm` (key: `source`) |
| Type: Daily | `bafyreiehsbrcsqdxuz63tbe6m3jqs6tkmnduxcyjq4bmw4pw54ybarmv4u` (key: `daily`) |

API 使用 `mcp__anytype__API-*` tools。需要時 ToolSearch 載入：search-space / create-object / update-object / get-object / list-objects。

### Tag IDs（select properties — **要傳 ID 不是 name**）

`status`:
- `seedling` → `bafyreigw3pkcl2d6hb2pc5644xyxwdasmmkptuer4xm6k47c2lvychlw6m`
- `budding` → `bafyreiebnmqpzujbcufp6ofbixaylun74r4e52rishz2u4mbuh6y66ngsi`
- `evergreen` → `bafyreifes7exmottu4lbrqswyvpj6nbkij3ui5d6sww44usz77znzwk5ey`

`kind`:
- `article` → `bafyreihlikgytbivoxzigmkrttcbp7gg44bmndmbjfyl4xqsffzf5xyo6m`
- `video` → `bafyreif6pxanppzmyqsox7sb3kdesm3dvfardfl2xvp2vbneblnfwogroa`
- `book` → `bafyreihhv6ebxiosy3wxfmtgi6gskdim5w7usto6tbxbzp5o6intxp7fu4`
- `podcast` → `bafyreiaja46e55ly5d43rjwvi3l2oaee7palya7gdwiegf4vgv7tjoqiqm`
- `paper` → `bafyreid3jzwvunccvqit44pmuou2z3hqvbx3g7nchhjpdhz3xjllizckjq`

### Topic properties
- `name` (object built-in) — Topic 主名
- `description` (text) — 一句話 summary。**注意：Anytype 自動把它 prepend 到 markdown body 顯示**，所以要寫得對人類讀者也有意義，不只是 LLM 比對用
- `status` (select: `seedling` / `budding` / `evergreen`) — 傳 tag ID
- `aliases` (text) — 別名，**逗號 + 空格分隔**：`PyTorch, pytorch, Py Torch`
- `last_merged_at` (date) — ISO 8601 with time, e.g. `2026-05-07T00:00:00Z`
- `relates_to` (objects) — Topic ↔ Topic
- `derived_from` (objects) — Topic → Source

主文用 object 的 **native markdown body**：
- create 時用頂層 `body` 欄位
- update 時用頂層 `markdown` 欄位（**會覆寫整個 body，merge 必須在 LLM 端先做完**）

### Source properties
- `name`, `description`, `url`, `author`
- `kind` (select) — 傳 tag ID
- `published_at`, `consumed_at` (date)
- `raw_excerpt` (text)
- `consumed_on` (objects) — Source → Daily

### Daily properties
- `name` — `YYYY-MM-DD`
- `description` (text)
- `entry_date` (date) — 主鍵
- `touched_on` (objects) — Daily → Topic[]

## Anytype API 行為（測過驗證）

1. **`select` 要 tag ID** 不是 name。tag IDs 在上面表格。
2. **`description` 自動 prepend 到 markdown body**（用戶在 page 開頭會看到）。
3. **`update-object` 用 `markdown` 欄位寫主文**，且是**覆寫不是 append**。merge 邏輯必須在 LLM 端先 get 舊內容、合併、再整段寫回。
4. **`search-space` 不查 `aliases` text 欄位**。只查 name + body content + description（case-insensitive）。需要找 alias 命中時必須額外做 client-side filter。
5. **`relates_to` 等 objects property 是 array — update 時整個替換**，要保留舊值就要先 get、append 新 ID、再 update 整個 array。
6. **`backlinks` 自動雙向**：A 設 relates_to=[B] 後，B 的 backlinks 自動含 A。但 backlinks 是泛型（含所有 incoming reference），沒有語意分類，所以 relates_to 仍要雙向 set 才能在 UI relates_to section 看到反向。
7. **`relates_to` / `derived_from` / `touched_on` 都會反映在內建 `links` 欄位**（outgoing object refs 全集合）。

## Core principles

1. **Search before write.** 任何寫入前都要先 `search-space` 找匹配。
2. **Merge body, don't append.** 新內容融入既有段落，必要時改寫舊句。**禁止** `## 2026-05-07: ...` 這種 timestamp 條目。
3. **Aliases prevent fragmentation.** 每次 capture 出現的別名都加進 `aliases`，避免「PyTorch / pytorch」分裂兩頁。
4. **Trace provenance.** 來自外部 Source 的內容在 body 對應段落末尾加 `— from <Source name>`，並設 `derived_from`。
5. **Reconcile, don't hide.** 若新資料跟舊 body 矛盾，並列兩個說法 + 出處，由 `/reconcile` 處理。

## Matching algorithm

Candidate topic name → 找 existing Topic：

1. `search-space` query=候選名, types=[topic key], limit=10
2. **必須**額外做 alias scan（search 不查 aliases）：
   - `search-space` query=空, types=[topic key], limit=200, sort=last_modified desc
   - client-side 過濾：`aliases` 字串 split by `, ` 後含候選名（case-insensitive）的 Topic
3. 合併兩組結果，去重
4. **LLM 判斷**是否同主題（name 同 / aliases 含 / description 語意同 → 是）
5. 沒有可信匹配 → 建新 Topic (`status: seedling`)
6. 有匹配 → PATCH（見 commands/capture.md）

> 註：步驟 2 的 limit=200 是 v1 妥協。Topic 數 > 200 時要改用分頁或自行索引。

## Daily linking

每個寫入操作都要把 touched 的 Topic 連到今日 Daily：

1. `search-space` query=YYYY-MM-DD, types=[daily key]（name match）
2. 找不到 → create Daily, name=YYYY-MM-DD, entry_date=today (ISO 8601)
3. 拿現有 `touched_on`，∪ 新 topic IDs（去重），update Daily 整個 array

## Status promotion rules

- 新建 Topic → `seedling`
- `seedling` 被 merge → `budding`
- `budding` → `evergreen` 由用戶手動（不自動升）

## Commands

`commands/` 子目錄有每個命令的詳細 flow。`~/.claude/commands/` 的同名 .md 是入口。

- `commands/capture.md` — 將自然語言想法 merge 進對應 Topic
- `commands/find.md` — 搜尋 Topic / Source / Daily 並分桶顯示
- _(待補)_ daily / ingest / connect / reconcile / review
