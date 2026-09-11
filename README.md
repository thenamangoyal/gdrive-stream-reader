# gdrive-stream-reader

A Claude Code skill that teaches Claude how to reliably read Google Drive content when the local **Google Drive for Desktop** ("File Stream") mount can't serve it.

If you've ever had Claude tell you "No such file or directory" on a Drive folder you can clearly see in your Finder, or watched it choke on a `.gdoc` file because that's just a JSON pointer and not the actual document, this skill fixes that. It hands Claude a deterministic playbook for reading Docs, Sheets, Slides, and folder shortcuts via the Anthropic Google Drive MCP, and for handling the over-25k-token "result too large" case.

## What it solves

Drive for Desktop is a **streaming mount**, not a copy of your Drive. Two things break naïve file reads:

1. **`.gdoc` / `.gsheet` / `.gslides` files are pointers, not documents.** They're tiny JSON files with a `doc_id` field. `cat`-ing them gets you metadata; the actual document body has to be fetched over the network.
2. **Folder shortcuts don't materialize until you open them in the Drive web UI.** A shortcut to "My Friend's Shared Folder" resolves as a symlink under `.shortcut-targets-by-id/<id>/...`, but `ls` returns `No such file or directory` until the folder gets opened in `drive.google.com`.

3. **Docs with tabs come back as one long body.** The Drive MCP returns every tab concatenated, each starting with a `# <tab title>  ` heading, and the `?tab=t.…` id in a Docs URL cannot be passed to it. Long multi-tab docs always trip the 25k-token ceiling, so the skill indexes the tabs from the saved file and reads them one at a time.
4. **Docs shared with a different Google account than the MCP is signed into** (personal vs Workspace) return "Requested entity was not found". The skill reads those through a second read-only `workspace-mcp` instance bound to that account (Docs API with `includeTabsContent`), never through a browser.

This skill walks Claude through the right MCP calls — `read_file_content`, `search_files`, and `download_file_content` — so it just works regardless of which case you hit.

## Prerequisites

1. **macOS or Windows** with [Google Drive for Desktop](https://www.google.com/drive/download/) installed and signed into your Google account.
2. **Claude Code** ([install](https://docs.claude.com/en/docs/claude-code/installation)) or any Claude harness that supports skills + MCP tools.
3. **Anthropic Google Drive MCP** connected to the same Google account that's signed into Drive for Desktop. In Claude Code:
   - Open Claude Code → `/mcp` to see configured MCPs.
   - Add the Anthropic Drive connector via `claude mcp add` (see [Claude Code MCP docs](https://docs.claude.com/en/docs/claude-code/mcp)) or via your `~/.claude.json` / `settings.json`.
   - Run any tool starting with `mcp__claude_ai_Google_Drive__` once to trigger the OAuth flow.
4. `jq` installed (`brew install jq` on macOS) for the chunked-read path. It comes pre-installed on most modern macOS systems.

## Installation

Skills in Claude Code live under `~/.claude/skills/`. Each skill is a directory containing a `SKILL.md` with a YAML frontmatter block.

### Option 1: Clone (recommended — easy to update)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/thenamangoyal/gdrive-stream-reader.git \
  ~/.claude/skills/gdrive-stream-reader
```

To update later:

```bash
cd ~/.claude/skills/gdrive-stream-reader && git pull
```

### Option 2: Curl one file

If you only want the skill content and don't care about updates:

```bash
mkdir -p ~/.claude/skills/gdrive-stream-reader
curl -fsSL https://raw.githubusercontent.com/thenamangoyal/gdrive-stream-reader/main/SKILL.md \
  -o ~/.claude/skills/gdrive-stream-reader/SKILL.md
```

### Verify

Open Claude Code in any directory and ask:

> "Is the gdrive-stream-reader skill loaded?"

Claude should confirm it can see the skill in its available list. Or check the source:

```bash
ls -la ~/.claude/skills/gdrive-stream-reader/SKILL.md
head -10 ~/.claude/skills/gdrive-stream-reader/SKILL.md
```

The frontmatter should start with `---` and include `name: gdrive-stream-reader`.

## Usage

You don't invoke this skill explicitly. The frontmatter description tells Claude when to load it on its own:

- Asking Claude to read a `.gdoc` / `.gsheet` / `.gslides` file.
- Asking Claude to "look inside" a Drive folder shortcut.
- Telling Claude that `ls` is returning "No such file or directory" on a Drive path.
- Telling Claude that a Drive doc is "too long" for it to read.

A typical session:

```
You:    Can you read drivedata/Q4-strategy.gdoc and summarize it?
Claude: [silently extracts doc_id, calls MCP read_file_content,
         summarizes the returned markdown body]
```

For the unmaterialized-folder case:

```
You:    What's in drivedata/Acme Customer Files/? It's a shortcut
        from a partner that I haven't opened yet.
Claude: [extracts folder id from the symlink target, calls
         search_files with parentId, lists the children,
         can recurse if you ask]
```

For the >25k-token doc case, the skill instructs Claude to use `jq` to decode the saved JSON and read in chunks — no need for you to do anything different.

## How it works (one paragraph)

A `.gdoc` file looks like this:

```json
{"":"WARNING! DO NOT EDIT...","doc_id":"1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789exampleID","resource_key":"","email":"..."}
```

The skill hands Claude a recipe: `cat` the file, pull `doc_id`, and call the Anthropic Drive MCP's `read_file_content` with that ID. The MCP fetches the document over OAuth and bypasses the local cache entirely. Same idea for folders: pull the folder id from the `.shortcut-targets-by-id/<id>/...` segment of the symlink target, then call `search_files` with `parentId = '<id>'`. Bulk operations parallelize via multiple tool calls in one message. Transient I/O errors get retried up to 3×.

The full procedure is in [`SKILL.md`](./SKILL.md), including a self-test checklist and the documented gotchas (e.g., `trashed` is *not* a supported query field — adding `and trashed = false` will error).

## Stress tests

Verified against a real Drive account with the following file types and edge cases (last additions 2026-09-11):

| Test | Path / setup | Expected | Result |
|---|---|---|---|
| `.gdoc` pointer extraction | `cat foo.gdoc` | JSON with `doc_id` | ✅ |
| `.gsheet` pointer extraction | `cat foo.gsheet` | JSON with `doc_id` | ✅ |
| `.gslides` pointer extraction | `cat foo.gslides` | JSON with `doc_id` | ✅ |
| MCP read of a Doc | `read_file_content` on a `.gdoc` id | markdown text body | ✅ |
| MCP read of a Sheet | `read_file_content` on a `.gsheet` id | markdown table body | ✅ |
| MCP read of Slides | `read_file_content` on a `.gslides` id | text content per slide | ✅ |
| Doc >25k tokens | a long-form ~60K-char doc | error pointing to saved file path | ✅ — `jq` decode then chunked read works |
| Unmaterialized folder | `ls drivedata/<shortcut>/` | `No such file or directory` (expected fail) | ✅ — `parentId` search returns the children |
| Bad query field | `search_files { query: "parentId='X' and trashed=false" }` | `Unsupported query field: trashed` | ✅ — gotcha documented |
| Direct PDF read | `Read drivedata/foo.pdf` | streaming mount fetches bytes | ✅ |
| Multi-tab Doc (7 tabs, 536k chars) | `read_file_content` on the doc id | all tabs in one saved body, `# <tab>  ` headings | ✅ — 2026-09-11, indexed with `grep -n '^# '` |
| Empty `{}` reply | first call on the same multi-tab doc | transient; second call returns the body | ✅ — retry documented in procedure E |
| Doc shared with the user's other Google account | `read_file_content` | `Requested entity was not found` (expected fail) | ✅ — second `workspace-mcp` instance (Docs API) documented in procedure G |
| Notion public page | `WebFetch` on `*.notion.site` | returns only "Notion" (expected fail) | ✅ — Chrome `get_page_text` documented in procedure H |

## Troubleshooting

**"Skill isn't being invoked even though I'm reading a `.gdoc`"**
Restart Claude Code. Skills are loaded at startup. If you cloned into the wrong path, `find ~/.claude/skills -name SKILL.md` will tell you.

**"MCP `read_file_content` returns 401 / unauthorized"**
The Drive MCP isn't OAuth'd, or it's authenticated to a different Google account than the one that owns / can see the doc. Run any Drive MCP tool in Claude Code and follow the OAuth prompt. Confirm with `mcp__claude_ai_Google_Drive__hf_whoami` (or the equivalent `whoami`-style tool if the MCP exposes one) which account it's connected to.

**"`Unsupported query field: trashed`"**
Remove `and trashed = false` from your `search_files` query. Trashed and spam files are excluded by default.

**"The MCP only gave me one tab / I need tab X"**
The MCP returns all tabs in one body; there is no per-tab read. Index the saved body with `grep -n '^# '` and slice by line range (procedure F). If only one heading appears, the doc really has one tab.

**"`Requested entity was not found` but I can open the doc"**
The doc is shared with a different Google account than the MCP is authenticated to. Register a read-only `workspace-mcp` instance for that account and use its `get_doc_content` (procedure G), or share the doc with the MCP account. Do not read Docs through a browser.

**"`jq: command not found`"**
`brew install jq` on macOS, or `winget install jqlang.jq` on Windows.

**"It works for me but not for my colleague"**
The MCP must be OAuth'd separately on each user's machine. The Drive for Desktop streaming mount also has to be installed locally; the skill won't help if your colleague is on a Linux box without Drive for Desktop (no `.gdoc` shortcut files exist there).

## Why a skill instead of a new MCP?

The flow only uses **existing tools** (`Bash`, `Read`, the Anthropic Drive MCP, optionally `jq`). It's not adding new tool primitives — it's teaching Claude the right sequence of tool calls. That's exactly what skills are for. Building this as an MCP would have required reinventing the auth flow and the export pipeline; not worth it.

If a future need arises (e.g., wanting tool-level retries, batched reads with structured output, or write support for `.gdoc`), an MCP would make sense. For read-only browsing, the skill is enough.

## Limitations

- **Read-only.** No support for editing or creating Drive content. The Anthropic Drive MCP exposes write tools (`create_file`, `update_page`) — use them directly when you need them; this skill does not wrap them.
- **No `resource_key` support.** Some old shared Drive items require a `resource_key`. The current Drive MCP does not pass one, so `read_file_content` may fail with permission errors. Workaround: open the doc in Drive UI once (which adds it to your "shared with me" properly), then re-try.
- **OS scope.** macOS is the primary target. Windows works for the MCP-based parts; the `cat`/`readlink` shell commands need PowerShell substitutes. Linux without Drive for Desktop has no `.gdoc` shortcut files at all and falls back to using `search_files` directly with the document title.
- **Drive Shortcut chains.** A shortcut to a shortcut isn't tested. Should still work because the symlink target resolves to a `.shortcut-targets-by-id/<id>/` path either way, and `parentId` search works on the same id.

## Contributing

PRs welcome. The skill content is a single Markdown file (`SKILL.md`) with YAML frontmatter. Keep it short, declarative, and decision-tree-shaped — don't turn it into a wall of prose.

If you find a new gotcha (especially a new "unsupported query field" or a case where the MCP behavior differs from the docs), please open an issue with a minimal repro.

## License

MIT — see [`LICENSE`](./LICENSE). Use freely.
