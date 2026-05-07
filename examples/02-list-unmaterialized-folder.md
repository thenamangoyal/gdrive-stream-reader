# Example 2: List a folder shortcut that hasn't been opened in Drive UI

You ask:

> What's inside `drivedata/Acme Customer Files/`? Someone shared this with me but I haven't opened it on the web yet.

You try `ls drivedata/Acme Customer Files/` yourself and get:

```
ls: drivedata/Acme Customer Files/: No such file or directory
```

Claude (with this skill loaded) does:

```bash
readlink "drivedata/Acme Customer Files"
# /Users/.../GoogleDrive-.../.shortcut-targets-by-id/<FOLDER_ID>/Acme Customer Files

# Pull the folder id:
readlink "drivedata/Acme Customer Files" \
  | sed -n 's:.*/\.shortcut-targets-by-id/\([^/]*\)/.*:\1:p'
# <FOLDER_ID>
```

Then calls:

```
mcp__claude_ai_Google_Drive__search_files {
  query: "parentId = '<FOLDER_ID>'",
  pageSize: 100,
  excludeContentSnippets: true
}
```

This returns the children directly from Drive over OAuth — no need for the local mount to materialize the folder.

If a child is itself a folder, Claude recurses with that child's `id`. If a child is a Google-native file (`mimeType` starts with `application/vnd.google-apps.`), Claude uses Procedure A (the `.gdoc` recipe) on its `id` to read the body.

## Common error if you get this wrong

Adding `and trashed = false` to the query — which seems sensible, since real Drive API does support it — errors out on the Anthropic Drive MCP:

```
Unsupported query field: trashed
```

Just leave it off. The MCP excludes trashed and spam by default.
