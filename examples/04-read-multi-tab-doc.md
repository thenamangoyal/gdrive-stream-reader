# Example 4: Read every tab of a multi-tab Google Doc

You ask:

> Read this product doc and all of its tabs: https://docs.google.com/document/d/1P4p…/edit?tab=t.0

Claude (with this skill loaded) does:

1. Takes the document id from the URL (the `tab=` part cannot be passed to the MCP).
2. Calls `mcp__claude_ai_Google_Drive__read_file_content { fileId: "1P4p…" }`. Because the doc has seven tabs and 536k characters, the reply is saved to `tool-results/…txt`.
3. Decodes and indexes the tabs:

```bash
jq -r '.fileContent' tool-results/mcp-claude_ai_Google_Drive-read_file_content-….txt > "$SCRATCH/product.txt"
grep -n '^# ' "$SCRATCH/product.txt"
#    1:# PRD  
#  151:# TDD — Browser Agent MVP  
#  646:# Competitive / market analysis  
#  973:# 09-07 User Intent Modeling POC  
# 2254:# 09-09 Conversation Transcript  
# 8992:# 09-10 PRD  
# 9033:# Narrative Memo (Old)  
```

4. Reads each tab by line range (`sed -n '151,645p'` or `Read` with `offset`/`limit`), stripping blank lines on the transcript tabs and checking with `md5` whether "Part 1" and "Part 2" of a pasted transcript are the same session before reading both.

## Gotchas seen on this doc

- The first `read_file_content` call returned a bare `{}`; the second returned everything. Retry once before concluding the doc is empty.
- Two sibling docs shared only with the user's Workspace account returned `Requested entity was not found` from the MCP (signed in to the personal account). The fix is a second read-only `workspace-mcp` instance bound to the Workspace account (procedure G); Google Docs are never read through a browser.
