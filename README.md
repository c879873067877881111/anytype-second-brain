# anytype-second-brain

`#anytype` `#claude-code` `#claude-skill` `#second-brain` `#llm-wiki` `#knowledge-management` `#mcp` `#karpathy-wiki`

An LLM-rewritten wiki for [Anytype](https://anytype.io/), built as a Claude Code skill.

Inspired by [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): every new piece of information **merges into existing Topic pages** instead of appending new notes. Contradictions stay visible, aliases prevent fragmentation, the wiki rewrites itself.

> Same idea as [eugeniughelbur/obsidian-second-brain](https://github.com/eugeniughelbur/obsidian-second-brain), but for Anytype's typed-object data model rather than Obsidian markdown files. Different storage layer → different schema design.

## Status

**WIP — 2/7 commands done.**

| Command | Status | Purpose |
|---|---|---|
| `/capture` | ✅ | Merge a thought into matching Topic(s) |
| `/find` | ✅ | Search Topic / Source / Daily |
| `/daily` | TODO | Today's Daily entry |
| `/ingest` | TODO | URL/video → multi-Topic merge + Source |
| `/connect` | TODO | Add `relates_to` between Topics |
| `/reconcile` | TODO | Surface and resolve contradictions |
| `/review` | TODO | Surface stale / due-for-review Topics |

## Schema

Three Anytype types:

- **Topic** 🌳 — the wiki page. `name`, `description`, `aliases`, `status` (seedling/budding/evergreen), `last_merged_at`, `relates_to`, `derived_from`. Body is native markdown.
- **Source** 📚 — external reference. `url`, `author`, `kind`, `published_at`, `consumed_at`, `raw_excerpt`, `consumed_on`.
- **Daily** 📅 — time-axis snapshot. `entry_date`, `touched_on` (Topics referenced that day).

Schema setup runs once via Anytype API (`mcp__anytype__API-create-type`/`create-property`).

## Install

Anytype must be running locally with the [official MCP server](https://github.com/anyproto/anytype-mcp) configured in Claude Code.

Clone and symlink into `~/.claude/`:

```sh
git clone <this repo> ~/projects/anytype-second-brain
ln -s ~/projects/anytype-second-brain/skills/anytype-second-brain ~/.claude/skills/anytype-second-brain
ln -s ~/projects/anytype-second-brain/commands/capture.md ~/.claude/commands/capture.md
ln -s ~/projects/anytype-second-brain/commands/find.md ~/.claude/commands/find.md
```

Then update `skills/anytype-second-brain/SKILL.md` with **your own** space ID, type IDs and tag IDs (the ones in there are the author's space — they will not work for you).

## Layout

```
skills/anytype-second-brain/
  SKILL.md              # shared resources, API behaviour notes, matching algorithm
  commands/
    capture.md          # full /capture flow (6 steps)
    find.md             # full /find flow (5 steps)

commands/
  capture.md            # slash-command entry point → reads SKILL.md + commands/capture.md
  find.md               # slash-command entry point
```

## Why an Anytype version

Anytype's typed objects + relations are a stronger substrate for the LLM-rewriting-wiki pattern than markdown files:

- `relates_to` is a first-class typed relation, not a `[[wiki link]]` string.
- Aliases live in a structured property, not the file body.
- `backlinks` are automatic and bidirectional.
- The LLM can `update-object` a single property without re-serialising the whole file.

The trade-off: `search-space` doesn't index alias text, so `/capture` and `/find` must do an extra alias scan client-side. The skill's `commands/*.md` files document this and other Anytype-specific quirks discovered during testing.

## License

TBD.
