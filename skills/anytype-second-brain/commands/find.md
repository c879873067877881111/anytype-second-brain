# /find flow

## Input

`$ARGUMENTS` = 關鍵字或自然語言查詢字串。

## Step 1 — Search

`mcp__anytype__API-search-space`:

```
space_id: <SKILL.md 的 Space ID>
query: $ARGUMENTS
types: ["topic", "source", "daily"]
limit: 20
sort: { property_key: "last_modified_date", direction: "desc" }
```

**為什麼 types 限三個**：space 裡可能有 Anytype 內建 type（書籤、文件、聊天等）混進結果，明確列我們關心的三個能去雜訊。

## Step 2 — Categorize

把回傳結果按 `type.key` 分三桶：
- `topic`
- `source`
- `daily`

每桶內保留 search 原本的順序（Anytype 已按 last_modified_date desc 排）。

## Step 3 — Alias filter（無條件做）

`search-space` **不查 `aliases` 欄位**（已測試確認）。任何時候都要補一輪 alias scan：

a. `mcp__anytype__API-search-space`:
   - `query`: ""（空字串）
   - `types`: `["topic"]`
   - `limit`: 200
   - sort 預設 last_modified desc

b. client-side 過濾：取每個 Topic 的 `aliases` text，split by `, ` 後 trim，檢查是否含 user query（case-insensitive substring）。

c. 把命中的 Topic 加進 Topic 桶，按 object id 去重。原 search 已命中的優先排前面，alias-only 命中的排後面標 `(via alias)`。

> Topic 數 > 200 時這個簡單實作會漏。v1 接受這個限制；之後可以分頁。

## Step 4 — Format output

每桶獨立段落，桶內為空就不顯示該段。

### Topics
```
🌳 <name> · <status> · merged <relative date>
   <description>
   aliases: <aliases>     ← 只在非空時顯示
```

### Sources
```
📚 <name> · <kind> · consumed <date>
   <description>
   <url>     ← 只在非空時顯示
```

### Dailies
```
📅 <name> · touched <N> topics
   <description>     ← 只在非空時顯示
```

**Relative date** 格式：`today` / `Nd ago` / `Nw ago` / `Nmo ago` / `<YYYY-MM-DD>`（超過 1 年）。

## Step 5 — Empty results

完全沒結果時：

```
找不到 "<query>" 相關內容。
試試 /capture <想法> 建立第一筆。
```

## Anti-patterns

- ❌ 列出 body 全文（太長，只列 description）
- ❌ 顯示所有 property（只挑該 type 的關鍵 3–4 個）
- ❌ 自作主張對結果做 LLM re-ranking（search 已排好，別添亂）
- ❌ types 留空導致內建書籤/檔案混進結果
- ❌ 對沒有 description 的 Topic 顯示 "undefined"（直接省略那行）

## Examples

### A. 找到結果

```
/find autograd
```

```
Topics
🌳 autograd · budding · merged today
   PyTorch 的自動微分系統，使用 dynamic graph
   aliases: Autograd

🌳 backpropagation · seedling · merged today
   反向傳播演算法，autograd 的概念基礎

Sources
📚 PyTorch Autograd Mechanics · article · consumed 3d ago
   Official PyTorch docs 對 autograd 內部機制的解釋
   https://pytorch.org/docs/stable/notes/autograd.html
```

### B. Alias 命中

```
/find pytorch
```

第一輪只找到一個 (name 大小寫不同)，但有 alias `pytorch` 的 Topic 補進來：

```
Topics
🌳 PyTorch · evergreen · merged today (via alias)
   ...
```

### C. 空結果

```
/find quantum chromodynamics
```

```
找不到 "quantum chromodynamics" 相關內容。
試試 /capture <想法> 建立第一筆。
```
