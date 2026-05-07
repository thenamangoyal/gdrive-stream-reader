# Example 1: Read a single Google Doc

You ask:

> Can you read `drivedata/Q4-strategy.gdoc` and tell me the headcount plan?

Claude (with this skill loaded) does:

```bash
cat "drivedata/Q4-strategy.gdoc"
# {"":"WARNING! DO NOT EDIT...","doc_id":"1Abc...XYZ","resource_key":"","email":"..."}
```

Then calls the MCP:

```
mcp__claude_ai_Google_Drive__read_file_content { fileId: "1Abc...XYZ" }
```

The MCP returns the document body as markdown. Claude reads it, finds the headcount section, and answers.

## Why this matters

Without the skill, Claude is likely to either:

- `cat` the `.gdoc` and report back the JSON pointer ("looks like an empty file").
- Try `Read` on the path and get `cat`'s 200-byte JSON output.
- Hallucinate content based on the filename alone.

The skill teaches it the one extra hop (extract `doc_id`, call MCP) that turns the pointer into the real document.
