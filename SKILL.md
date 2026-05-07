---
name: gdrive-stream-reader
description: Read Google Drive content (Docs, Sheets, Slides, and folders) when the local Google Drive for Desktop streaming mount cannot serve it. Use this skill whenever the user asks you to read a `.gdoc` / `.gsheet` / `.gslides` file, when an `ls` on a `drivedata/...` symlink returns "No such file or directory", when a path resolves under `.shortcut-targets-by-id/`, or whenever the user asks about extracting Google Drive content on macOS or Windows with the Drive for Desktop streaming mount. Also use it for documents whose content exceeds the Drive MCP tool's inline 25k-token reply ceiling (the tool will save the content to disk for chunked reading).
---

# Google Drive streaming reader

This skill teaches Claude how to reliably read Google Drive content when the local Drive for Desktop ("File Stream") mount can't materialize it. The core insight is that `.gdoc` / `.gsheet` / `.gslides` files are **JSON pointers**, not the document body, and that folder shortcuts only materialize on the local mount once they have been opened in the Drive web UI.

The skill relies on the **Anthropic Google Drive MCP** (`mcp__claude_ai_Google_Drive__*` tools). If those tools are not present in the current session, this skill does not apply — direct the user to set up the Drive MCP first (see the `prerequisites` section in `README.md` of the source repo).

## When to invoke this skill

Trigger on any of:

1. The user asks to read or analyze a `.gdoc`, `.gsheet`, or `.gslides` file path.
2. `ls`, `find`, or `Read` on a path under `drivedata/`, `~/Library/CloudStorage/GoogleDrive-…`, or any `.shortcut-targets-by-id/<id>/…` returns `No such file or directory` even though the symlink itself resolves.
3. The MCP tool `mcp__claude_ai_Google_Drive__read_file_content` returns the "result exceeds maximum allowed tokens" error and saves the content to a `tool-results/...txt` file.
4. The user describes a Drive folder shortcut they "haven't opened yet" or that's not visible locally.
5. Any session where files in `drivedata/` or a Drive-mounted directory need to be read in bulk.

## Decision tree

```
Need to read something from Google Drive on a streaming mount?
│
├── It's a .gdoc / .gsheet / .gslides file
│   └── Use procedure A (pointer-extract → read_file_content)
│
├── It's a regular file (PDF / .docx / image / video) under drivedata/
│   └── Use procedure D (Read directly; the streaming mount serves bytes on demand)
│
├── It's a folder I can't `ls` (returns ENOENT but the symlink resolves)
│   └── Use procedure B (readlink → search_files with parentId)
│
├── I called read_file_content and got the >25k-token error
│   └── Use procedure C (jq decode + chunked sed reading)
│
└── I got a transient I/O error
    └── Use procedure E (retry up to 3× with brief backoff)
```

## Procedure A: Read a `.gdoc` / `.gsheet` / `.gslides`

The shortcut file on disk is a JSON pointer. Extract the `doc_id` and ask the MCP for the content.

```bash
cat "<path>.gdoc"
# {"":"WARNING! DO NOT EDIT...","doc_id":"<DOC_ID>","resource_key":"...","email":"..."}
```

Then call (in this very order):

```
mcp__claude_ai_Google_Drive__read_file_content {
  fileId: "<DOC_ID>"
}
```

This works for all three Google-native MIME types (Docs, Sheets, Slides). The MCP returns a natural-language text representation:

- `.gdoc` → markdown text.
- `.gsheet` → markdown table per sheet.
- `.gslides` → text content per slide.

If the user wants a different format (a real `.pdf` of a Slides deck, a real `.xlsx` of a Sheet), use:

```
mcp__claude_ai_Google_Drive__download_file_content {
  fileId: "<DOC_ID>",
  exportMimeType: "application/pdf"   // or vnd.openxmlformats-officedocument.wordprocessingml.document
}
```

The download tool returns a base-64 encoded payload.

## Procedure B: List a folder shortcut whose target isn't materialized locally

Drive for Desktop only fetches the bytes of folders the user has actually opened in the Drive UI. A shortcut you've never opened resolves as a symlink on disk but `ls` returns `No such file or directory`.

The folder id is encoded in the symlink target (the path component just under `.shortcut-targets-by-id/`):

```bash
readlink "drivedata/<Shortcut Folder>"
# /Users/.../.shortcut-targets-by-id/<FOLDER_ID>/<Shortcut Folder Name>

# Extract the folder id:
readlink "drivedata/<Shortcut Folder>" \
  | sed -n 's:.*/\.shortcut-targets-by-id/\([^/]*\)/.*:\1:p'
```

Then list the children via Drive search:

```
mcp__claude_ai_Google_Drive__search_files {
  query: "parentId = '<FOLDER_ID>'",
  pageSize: 100,
  excludeContentSnippets: true
}
```

**Critical:** do **not** add `and trashed = false` — `trashed` is *not* a supported query field on this MCP and the call will error out (`Unsupported query field: trashed`). The default already excludes trashed and spam.

For folders with more than 100 children, paginate by passing the returned `nextPageToken` back in the next call as `pageToken`.

To recurse into a child folder, use the same procedure with the child's `id`. Mixing depth-first and breadth-first traversal is fine; the MCP doesn't preserve order.

When you encounter a Google-native file in the listing (mimeType starts with `application/vnd.google-apps.`), feed its `id` to procedure A.

## Procedure C: Decode a doc that exceeded the 25k-token reply ceiling

When `read_file_content` is called on a long doc, the MCP doesn't return the body inline. Instead it returns an error like:

```
Error: result (63,961 characters) exceeds maximum allowed tokens.
Output has been saved to /Users/.../tool-results/mcp-claude_ai_Google_Drive-read_file_content-<id>.txt.
Format: JSON with schema: {fileContent: string}
```

The data is **already on disk** in JSON form. Do not re-call the MCP. Decode and chunk-read:

```bash
jq -r '.fileContent' /path/to/tool-results/...txt > /tmp/<doc-name>.txt
wc -lc /tmp/<doc-name>.txt
```

For very long docs, the file may have one logical line that's tens of KB long. Use `sed -n '<start>,<end>p'` for line-based chunking:

```bash
sed -n '1,500p' /tmp/<doc-name>.txt
sed -n '500,1000p' /tmp/<doc-name>.txt
```

Or read with the `Read` tool's `offset` and `limit` parameters (preferred — it handles cat-n line numbers).

If `Read` reports "File content exceeds maximum allowed tokens" even on the decoded `/tmp/<doc-name>.txt`, fall back to `sed`/`head` slicing.

## Procedure D: Plain files (PDFs, images, .docx, video)

Just `Read` the file directly through the streaming mount:

```
Read { file_path: "/Users/.../drivedata/<file>.pdf" }
```

The mount fetches bytes on demand. PDFs are returned as multi-page documents that Claude can read inline (up to 20 pages without `pages: "1-N"`).

For very large binary files, use `mcp__claude_ai_Google_Drive__download_file_content` if you need to send them somewhere (it returns base-64), or just operate on the local path.

## Procedure E: Retry on transient streaming errors

Drive for Desktop sometimes returns:

- `Operation not permitted`
- `Input/output error`
- `Resource temporarily unavailable`
- An empty read on a file that should exist
- A 5xx from the MCP

These are usually streaming hiccups, not real failures. Wait briefly (1–5 seconds is fine) and retry up to **3 times** before declaring the file missing. Never "fix" a transient error by deleting, recreating, or renaming the file — that destroys the user's data.

If after 3 retries the read still fails, surface the error to the user with the original error string and the file path; do not silently swallow it.

## Bulk reads — parallelize via concurrent tool calls

When you need to read many `.gdoc` files, do not loop sequentially. Issue all the `read_file_content` calls in **a single message with multiple tool blocks**. The MCP processes them concurrently. Example: reading 6 `.gdoc` files takes roughly the time of one slow read instead of six round-trips.

For folder traversal, the breadth-first version (collect all child IDs at one level, then issue all next-level reads in parallel) scales much better than depth-first.

## Cross-checking the user's `doc_id`

If a `.gdoc` pointer's `doc_id` looks suspicious (wrong length, wrong shape, doesn't start with `1`), confirm by also running:

```
mcp__claude_ai_Google_Drive__get_file_metadata {
  fileId: "<DOC_ID>",
  excludeContentSnippets: true
}
```

This returns title, owner, MIME type, size, and modified time without fetching the body. Useful for sanity checks.

## Path discovery if you don't have a path to start from

If the user says "find my X document" without giving a path, search Drive directly:

```
mcp__claude_ai_Google_Drive__search_files {
  query: "title contains '<keyword>' and mimeType = 'application/vnd.google-apps.document'",
  pageSize: 20
}
```

Supported query fields per the MCP's docs: `title`, `fullText`, `mimeType`, `modifiedTime`, `viewedByMeTime`, `createdTime`, `parentId`, `owner`, `sharedWithMe`. Do **not** use `trashed` (will error).

## Edge cases / things that still fail

- **`resource_key` requirement.** Some old shared docs require a `resource_key` (visible in the `.gdoc` JSON). The current MCP `read_file_content` tool does not accept a `resource_key` parameter. If a doc that requires one fails to read, the user has to open it in Drive UI first to convert it; there is no skill-side fix.
- **Files in another user's "Shared with me"** that have not been added to the user's own Drive. `search_files` with `sharedWithMe = true` finds them, but `read_file_content` may still fail with permission errors. Direct the user to add the file to their Drive.
- **Comment-mode and suggestion-mode artifacts.** `read_file_content` returns the rendered text, not tracked changes or comments. Use `download_file_content` with `exportMimeType: "application/vnd.openxmlformats-officedocument.wordprocessingml.document"` if comments matter.
- **`download_file_content` returns base-64.** If the user wants to view it locally, write it to disk yourself (`Write` after base64-decoding); the tool does not save it for you.
- **Windows.** Drive for Desktop on Windows uses `G:\My Drive\…` paths instead of macOS `~/Library/CloudStorage/…`. The `.gdoc` JSON format and the `.shortcut-targets-by-id` directory both exist on Windows, so the MCP-based parts of this skill work identically; only the local `cat`/`readlink` step needs OS-appropriate substitutes (`type` and `Get-Item -Type SymbolicLink` in PowerShell).

## What this skill does NOT do

- It does not replace the Drive MCP. The MCP is a hard prerequisite.
- It does not bypass auth. If the user is signed into a different Google account in Drive for Desktop than the one the MCP is OAuth'd to, the doc IDs may be valid on the local mount but unreadable via MCP. Confirm both sides are the same account before troubleshooting further.
- It does not write to Drive. The MCP tools used here are read-only (`read_file_content`, `download_file_content`, `search_files`, `get_file_metadata`). Writing tools (`create_file`, `update_page`) are out of scope for this skill — use them directly when needed.

## Self-test (when the user asks "is this skill working?")

Run the following four checks. All four should succeed on a healthy setup:

```bash
# 1. Pointer extraction
cat "drivedata/<some-file>.gdoc" | jq -r .doc_id
```

```
# 2. MCP read on a small doc
mcp__claude_ai_Google_Drive__read_file_content { fileId: "<DOC_ID>" }
```

```
# 3. parentId search on an unmaterialized folder
mcp__claude_ai_Google_Drive__search_files { query: "parentId = '<FOLDER_ID>'", pageSize: 5 }
```

```bash
# 4. Direct PDF Read on the streaming mount
Read { file_path: "drivedata/<some-file>.pdf" }
```

If any of (1)–(4) fails, the diagnosis is:

- (1) fails → not a real `.gdoc` pointer (or `jq` not installed; `brew install jq`).
- (2) fails with auth error → MCP not OAuth'd, or wrong Google account.
- (2) fails with "exceeds maximum allowed tokens" → that's success in disguise; use Procedure C.
- (3) fails with `Unsupported query field` → user has added `and trashed = false` to the query (do not).
- (4) fails with ENOENT → see Procedure E (retry); if still failing, see Procedure B (folder isn't materialized).
